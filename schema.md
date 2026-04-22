# Schema `v5`

<a align="center" href="https://www.notion.so/Script-Schema-32f26c5e63ba8068b513e8403bff413c">Notion Page</a>

## File Structure

```json
/*
	GameScript.json
*/

{
  "version": 5,
  "script": [
    { /* script entry */ }
  ]
}
```

## Script Entry

<aside>
<img src="/icons/alert_green.svg" alt="/icons/alert_green.svg" width="40px" />

A Script Entry represents a unit of conversation or event in the game.
A Script is a list of ordered Script Entries. An example is shown above.

</aside>

Everything follows the general format of:

```json
{
  "uid": "unique-id",
  "type": "entry-type",
  // ... other fields depending on entry type
}
```

The following entry types are valid:

```
boss
pig
senator
player

minigame
decision
branch

day
end

link
```

### Entry Types

#### **Boss / Pig / Senator / Player**

Represents a single message sent by the the Boss / Pig / Senator / Player.

```json
{
  "uid": "unique-id",
  "type": "senator",
  "effect": 0.0, // can be omitted, default to 0
  "msg": "This is what the senator would say.",
  "next": "uid-of-next-entry"
}
```

The `type` field should be one of

```jsx
boss
pig
senator
player
```

**v5 changes**

- `effect`: messaging entries now can have suspicion adjustment applied after

#### **Minigame**

Represents a message that is linked to open a minigame (`gameId`)

```json
{
  "uid": "unique-id",
  "type": "minigame",
  "msg": "This is the link to the minigame etc.",
  "direct": false, // can be omitted, default is false
  "no-bid": false, // can be omitted, default is false
  "gameId": 0,
  "winBid": 1, // can be omitted, default is 1
  "loseBid": 0, // can be omitted, default is 0
  "winEffect": 0.0, // can be omitted, default to 0
  "loseEffect": 0.0, // can be omitted, default to 0
  "next": "uid-of-next-entry"
}
```

**v3 changes:**

- `direct`: if true, launches the minigame in direct mode.
    - In direct mode, the minigame is launched immediately without sending a message and waiting for user interaction.
- `no-bid`: if true, launches the mini-game but will not emit a bid on game end

**v4 changes:**

- `winEffect`: the effect if the game is won
- `loseEffect`: the effect if the game is lost

#### **Decision**

Represents a response decision that is presented to the player.

Messages in Decision(s) are implicitly specified to be sent by the Player.

```json
{
  "uid": "unique-id",
  "type": "decision",
  "options": [
    {
      "msg": "This is what I would send for this option.",
      "effect": 0.7, // the effect of this decision on the game state
      "bid": 0 // unique within decision, can be omitted (auto increment from 0)
    },
    // ...
  ],
  "next": "uid-of-next-entry"
}
```

#### **Branch**

Represents a branching event.

```json
{
  "uid": "unique-id",
  "type": "branch",
  "branches": [
    {
      "bid": 0, // unique within branch
      "next": "uid-of-next-entry"
    },
    // ...
  ]
}

{
  "uid": "unique-id",
  "type": "branch",
  "branches": [
    "uid-of-next-entry", // implicit bid: 0
    "uid-of-next-next-entry", // implicit bid: 1
    // ...
  ]
}
```

`branches` accept either

1. An array of objects of type `{bid:int, next:str}` or
2. An array of strings.
Each string is a shorthand for `{bid: (auto-incremented), next: string}`

but not a mix of both.

#### **Day**

Represents the end of a day.

```json
{
  "uid": "unique-id",
  "type": "day",
  "next": "uid-of-next-entry"
}
```

#### **End**

Represents the end of the game.

```json
{
  "uid": "unique-id",
  "type": "end"
}
```

#### Link

Allows linking one script to another.

```json
{
  "uid": "unique-id",
  "type": "link",
  "target": "nextScript" // this would refer to nextScript.json
}
```

# Data Specs

## Message Strings `msg`

Messages are presented/saved as regular strings, with one quirk: strings can embed tags for specific behaviors; the tags are as follows:

- `<d>deletion text</d>` **Deletion Tag**: Same as the idea strikethrough used in the Google Docs script. This tag signals that the enclosed text should be animated as typed it, then deleted.

![Example excerpt from Day 3 in the script](attachment:975ac594-32c9-43f9-80c7-8bb1f1cc6e5b:image.png)

Example excerpt from Day 3 in the script

```json
 "msg": "<d>Oh now this sounds interesting!</d> I can definitely help you with that! What’s the document and what do you want me to change?"
```

## Reserved UIDs

UIDs starting with `iuid-` are reserved for internal use, and should not be used in the script as `uid` values.

## Branch ID `bid`

Branch IDs allow for deferred branching. For example, you can have a `decision` entry with options that have `bid`, and then later in the script have a `branch` entry that specify the next entry for each `bid`. This allows you to separate the content of the options and the structure of the branches.

Mixing entries with omitted `bid` and entries with specified `bid` in the same `decision` or `branch` is not allowed, the parser can throw.

`bid` entries are emitted by `type:"decision"` and `type:"minigame"`, and consumed by `type:"branch"`. Note that emitted entries that are not consumed will remain in game state until they are consumed, watch out for bleeding of `bid` values between different decisions and branches.

An example legal usage (with legal omission) of emitters/branches/bids is as follows:

```json
{ "type": "decision", "options": [
  { "msg": "Accept", "bid": 0 },
  { "msg": "Decline", "bid": 1 }
]},
{ "type": "senator", "msg": "Interesting choice..." },
{ "type": "branch", "branches": ["accept-node", "decline-node"] },
{ "uid": "accept-node", "type": "senator", "msg": "Welcome aboard.", "next": "after-decision" },
{ "uid": "decline-node", "type": "senator", "msg": "A shame. Good day." },
{ "uid": "after-decision", "type": "senator", "msg": "Now, let's continue..." }
```

## Omissions

All [Script Entry](https://www.notion.so/Script-Schema-32f26c5e63ba8068b513e8403bff413c?pvs=21) types can omit the `next` field (if one exists), in which case `next` will be inferred to be the entry after it. Behaviorally, the game will automatically jump to the next entry in the script.

All types can omit the `uid` field, a unique value will be generated internally. However, this also means that you cannot refer to this entry in the `next` field of other entries.

For example, the following is valid

```json
{
  "type": "senator",
  "msg": "This is what the senator would say.",
}
```
