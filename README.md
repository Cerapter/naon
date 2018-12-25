# Suggestions for the new AO netcode (2.7+) #

The following document describes a suggestion for the netcode rework 
for the upcoming 2.7 version of Attorney Online. The document itself
is mostly based on the [AO2 Protocol][ao2protocol], but also takes
inspiration from the [AO3 / AC Protocol][ao3protocol] as well. It is
suggested that the reader is familiar with at least the former, but
familiarity with the latter does not hurt, either, as the document
takes most of its ideas from that, except fitting it to AO2 where
possible or necessary.

As it was already agreed upon, the new networking system will use
Websockets, and its messages will be JSON documents.

## Table of Contents ##

- [Legend](#user-content-legend)
- [Packet setup](#user-content-packet-setup)
- [Handshake](#user-content-handshake)
  - [`HI`](#user-content-establishing-connection-hi)
  - [`SD`](#user-content-send-server-details-sd)
  - [`JS`](#user-content-join-server-js)
  - [`JR`](#user-content-server-join-result-jr)
  - [`DC`](#user-content-disconnect-dc)
  - [`KK`](#user-content-kick-kk)
  - [`KB`](#user-content-ban-kb)
- [Loading](#user-content-loading)
  - [`AL`](#user-content-request-and-get-asset-id-list-al)
  - [`AR`](#user-content-request-and-get-repository-list-ar)
  - [`LR`](#user-content-loading-ready-lr)
- [User profiles](#user-content-user-profiles)
  - [`SUP`](#user-content-set-user-profile-sup)
  - [`GUP`](#user-content-get-user-profiles-gup)
- [Messaging](#user-content-messaging)
  - [`IC`](#user-content-ic-messages-ic)
  - [`CT`](#user-content-ooc-messages-ct)
- [In-game](#user-content-in-game)
  - [`CC`](#user-content-change-character-cc)
  - [`MC`](#user-content-music-mc)
  - [`BG`](#user-content-change-background-bg)
  - [`PR`](#user-content-pair-up-request-pr)
  - [`PL`](#user-content-get-list-of-pairs-pl)
  - [`CA`](#user-content-check-character-availability-ca)
  - [`CH`](#user-content-keep-alive-ch)
  - [`MD`](#user-content-login-as-moderator-md)
  - [`ZZ`](#user-content-call-moderator-zz)
- [Area handling](#user-content-area-handling)
  - [`GA`](#user-content-get-list-of-areas-ga)
  - [`CM`](#user-content-become-an-owner-of-an-area-cm)
  - [`UM`](#user-content-give-up-ownership-of-an-area-um)
  - [`EA`](#user-content-edit-area-ea)
  - [`JA`](#user-content-join-area-ja)
  - [`MA`](#user-content-move-to-area-ma)
  - [`IA`](#user-content-invite-to-area-ia)
  - [`UA`](#user-content-uninvite-from-area-ua)

## Legend ##

- **`AO` / `AO2`:** Attorney Online 2
- **`NAON`:** new AO netcode -- a shorthand for this very suggestion.
- **`...`:** the argument list continues with the pattern established
before it.

[*Back to TOC*][toc]

## Packet setup ##

By default, every packet should begin with a *header:* a short string
that identifies it.

Further, it should be noted that while in this document, the pieces of
code are in TypeScript, this is only for easier readability -- the
networking itself should JSON, as it has been said before.

[*Back to TOC*][toc]

## Handshake ##

[*Back to TOC*][toc]

### Establishing connection: `HI` ###

*Client -> Server*    
*Response:* `SD`

#### Format: ####

```typescript
  {
    header: "HI",
    client: String,
    ver: String,
    vver: String
  }
```

The user's client introduces itself to the selected server.

The `client` argument contains the name of the client the user is
currently using.
In Vanilla, this would be `"Attorney Online"`, but servers with custom
clients could name them here.

The `ver` argument tells the server the version of the aforementioned
client.

Finally, the `vver` tells the server which version of the Vanilla
client is the current client most compatible with.

The server can use these pieces of information to reject, approve,
limit or warn the joining client.

Implicitly, Vanilla is most compatible with itself, so in Vanilla's
case, the two versions should be one and the same.

[*Back to TOC*][toc]

### Send server details: `SD` ###

*Server -> Client*

#### Format: ####

```typescript
  {
    header: "SD",
    name: String,
    desc: String,
    players: Number,
    maxplayers: Number,
    protection: 0 | 1 | 2
  }
```

Sent in response to a `HI` packet. The server explains some basic
details about itself.

Most arguments are self-explanatory, however, the `protection` argument
can take three different values:
- `0`: open. Anyone can join.
- `1`: password or spectate. A user can only join as a player if they
give the correct password. 
However, they can choose to spectate if they do not know it.
- `2`: password needed. The only way to join the server, player or
spectator, is if the user knows the password.

[*Back to TOC*][toc]

### Join server: `JS` ###

*Client -> Server*    
*Response:* `JR`

#### Format: ####

```typescript
  {
    header: "JS",
    spectate: Boolean,
    password: String
  }
```

The client sends a request to join the server.

If `spectate` is true, the client wants to join as a spectator.
Else, it wants to join as a player.

The `password` string contains the client's guess at the server's
password, assuming it is passworded.

[*Back to TOC*][toc]

### Server join result: `JR` ###

*Server -> Client*
*Response:* `AL`

#### Format: ####

```typescript
  {
    header: "JR",
    result: 0 | 1 | 2 | 3 | 4 | 5,
    message: String
  }
```

Sent to the client upon an attempt the join.

The `result` argument can be any of the following:
- `0`: successful join.
- `1`: server full.
- `2`: spectating is not allowed.
- `3`: wrong password.
- `4`: banned.
- `5`: rejected for other reasons.

The `message` argument details a short message that goes alongside
the result.
Though only applicable with result `5`, nothing should stop the server
from sending a message alongside any of the other results.

[*Back to TOC*][toc]

### Disconnect: `DC` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
{
  header: "DC",
  message: String
}
```

*From server:*
```typescript
{
  header: "DC",
  id: Number,
  message: String
}
```

If sent by the client, formally closes the connection to the server,
while giving a reason for leaving in the `message` argument.

If sent by the server, it announces the disconnection of the client
with the given ID, and their reason to leave in `message`.

In both cases, `message` is optional, and will default to `"User
disconnected"` if not specified or left empty.

To actually force people to quit, see `KK` and `KB`.

[*Back to TOC*][toc]

### Kick: `KK` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
{
  header: "KK",
  id: Number,
  message: String
}
```

*From server:*
```typescript
{
  header: "KK",
  message: String
}
```

If sent by the client, requests that the server kick the player with
the given ID, and the reason in `message`.

If sent by the server, kicks the client it is sent to.
It should force the client to disconnect, though the server should
make sure to cease connection as well, to avoid trickery.

In both cases, `message` is option, and defaults to `""`.
Keeping the default value does not display a reason.

[*Back to TOC*][toc]

### Ban: `KB` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
{
  header: "KB",
  id: Number,
  message: String
}
```

*From server:*
```typescript
{
  header: "KB",
  message: String
}
```

If sent by the client, requests that the server ban the player with
the given ID, and the reason in `message`.

If sent by the server, bans the client it is sent to.
It should force the client to disconnect, though the server should
make sure to cease connection as well, to avoid trickery.
Further, the server should keep the player from rejoining.

In both cases, `message` is option, and defaults to `""`.
Keeping the default value does not display a reason.

[*Back to TOC*][toc]

## Loading ##

*Attorney Online 2.7* will use an asset system wherein it will the
client that handles the downloading and unpacking of content upon
joining a server, rather than the user manually doing so.    
Of course, manual installation will still be possible, but should not
be the preferred way of handling content.

Anything that needs to be said on the matter is available in a bit
more detail [here][ao2.7assets].

[*Back to TOC*][toc]

### Request and get asset ID list: `AL` ###

*Client -> Server*    
*Server -> Client*
*Response (one or the other):* `AR`, `LR`

#### Format: ####

*From client:*
```typescript
{
  header: "AL"
}
```

*From server:*
```typescript
{
  header: "AL",
  ids: [
    String,
    String,
    String,
    ...
  ]
}
```

If sent by the client, requests the IDs of all used assets on the
server.

If sent by the server, returns said IDs.

After the list has arrived back to the client, it should query its
database to determine which assets are missing.

It should then look for those missing assets in its currently
registered repositories, and if they are not found, request the server
for additional repository suggestions.

If all assets can be found, signal to the server that the client is
ready with the download.

[*Back to TOC*][toc]

### Request and get repository list: `AR` ###

*Client -> Server*    
*Server -> Client*
*Response:* `LR`

#### Format: ####

*From client:*
```typescript
{
  header: "AR"
}
```

*From server:*
```typescript
{
  header: "AR",
  repos: [
    String,
    String,
    String,
    ...
  ]
}
```

If sent by the client, the client could not find all the assets in
its default repositories, and requests the server for others to
look into.

If sent by the server, returns the list of URLs to look into.

The URLs can be of two kind:
- "Big" repositories, which follow the format of `URL/assets` to get
the asset list, and `URL/assets/{id}` to download a specific asset
with a given ID.
- "Mini" repositories, where instead the `URL` will point to a
manifest file, that contains a list of asset IDs, and where to
download them from.

After the asset list has been obtained, the client should try to
get all the assets. Should it fail to find some, it should alert the
user so.

Either way, when all possible assets are downloaded, the client should
signal to the server that it is ready with the loading.

[*Back to TOC*][toc]

### Loading ready: `LR` ###

*Client -> Server*

#### Format: ####

```typescript
  {
    header: "LR"
  }
```

Simply tells the server that the client has all the assets, or
downloaded all the assets it could.

In response to this, the server should "officially" add the client,
putting them in the default area, and sending all the necessary
info about the gamestate.

[*Back to TOC*][toc]

## User profiles ##

User profiles are a short little blurb of information about users,
presented in the style of profiles like in the original *Ace Attorney*
games.

Users should be able to set up their player profiles before joining
a server, and change it inside a server.

A user profile displays the user's currently selected character,
their preferred OOC name, and a short description of them.

Users may decide to leave these fields empty, of course.

User profiles can be presented like evidence or profiles in the
*Ace Attorney* games, and this results in a notification for the user
whose profile was presented -- somewhat similar to callwords from
pre-2.7 in function.

[*Back to TOC*][toc]

### Set user profile: `SUP` ###

*Client -> Server*    
*Response:* `GUP`

#### Format: ####

```typescript
  {
    header: "SUP",
    name: String,
    desc: String
  }
```

Sends the user's preferred OOC name and a short description of them.

Both arguments are optional. If they are left out, or left empty 
(`""`), the server should assign the user a unique (server-wide) name,
and the description "???".

[*Back to TOC*][toc]

### Get user profiles: `GUP` ###

*Server -> Client*

#### Format: ####

```typescript
  {
    header: "GUP",
    users: [
      {
        id: Number,
        name: String,
        desc: String,
        charid: Number,
        area: Number
      },
      ...
    ]
  }
```

Gives the client a list of user profiles from the server, from all
areas.

If a client gets a `GUP` packet, it should delete all previously
stored user profiles.

A `User Profile` consists of:
- `id`: the user's ID on the server.
- `name`: the user's OOC name.
- `desc`: the user's description.
- `charid`: the ID of the character the user is currently playing as.
- `area`: the ID of the area this user is currently in.

[*Back to TOC*][toc]

## Messaging ##

[*Back to TOC*][toc]

### IC messages: `IC` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```typescript
  {
    header: "IC",
    pieces: [
      ICPiece,
      ICPiece,
      ICPiece,
      ...
    ]
  }
```

In AO 2.7+, IC messages would be made up of `ICPiece`s. An `ICPiece`
is a short set of instructions that makes up the full content of an
actual IC message. These would be *roughly* equivalent to the pre-
2.7 pieces of an `IC` packet.

By default, the *complexity* of an `IC` packet is determined by the 
amount of `ICPiece`s it is made up of. Servers, however, can decide to
assign different 'weights' to different `ICPiece`s.

If an `ICPiece` is said to *default to* something, it should be
interpreted as if an `ICPiece` of that type with that default value
were implicitly put before any other `ICPiece` in the IC message.
In practice, this system may be replicated, though it would most
likely be implemented by the IC message displaying system having some
necessary defaults.

Unless specified otherwise, every argument is mandatory.

Here is a non-exhaustive list of the possible `ICPiece` variants,
and their arguments:
- `"emote"`
  - `emote: String`: the emote to change to.    
  No defaults.    
  Invalid emotes are displayed as a 'missingno' as with pre-2.7.
  - `anim: String`: the name of the animation that should play once,
  without looping, before the emote itself appears.    
  No defaults.    
  Optional.    
  As with `emote`, invalid animations are to be displayed as a
  'missigno', and be considered immediately finished, having no actual
  animation.
  - `stall: Boolean`: if `true`, the client should not continue
  interpreting the IC message further until the animation has played
  completely.    
  If `true`, and the animation does not finish by the time the IC
  message is over, the IC message is nevertheless considered finished,
  and a new one may be interpreted.    
  If `false`, play the animation, but continue interpreting the IC
  message. If need be, default to the last defined `emote`.    
  If `false`, and the animation is playing, and a new `"emote"` is
  defined before the animation has finished, switch to the new one.    
  If `anim` exists, so should this. Otherwise, it should not appear.

  With one exception (see below), **this must be the first `ICPiece`
  in an IC message.**
- `"text"`
  - `text: String`: the text to insert into the message, that will be
  displayed in the message box.    
  No defaults.    
  An empty text `ICPiece` is not considered to exist, and should be 
  pruned from the IC message itself.
  - `center: Boolean`: if `true`, the message will appear centred.    
  If another `"text"` `ICPiece` appears later on, that has a `center`
  with the opposite value, that one is ignored.    
  The first `"text"` piece determines whether the entire text is
  centered or not.
- `"speed"`
  - `speed: Number`: the speed at which any following `"text"` `ICPiece`
  should be displayed, in percentages.    
  Defaults to `100`.    
  Larger values imply faster speeds.    
  Sanity checks are heavily encouraged here, both on client and server
  side.    
  A value of `0` should be rejected outright.    
  If no `"text"` `ICPiece` comes after this piece, it should be pruned
  from the IC message.
- `"color_theme"`
  - `color: Number`: the colour of any following `"text"` `ICPiece`s,
  where the colour itself is determined by user's current theme.    
  Defaults to `0`.    
  If no `"text"` `ICPiece` appears after this piece, it should be 
  pruned from the IC message. The acceptable values are:
    - `0`: default text colour. The default theme's white.
    - `1`: key text colour, cross-examination body text colour. 
    The default theme's green.
    - `2`: important fact colour, testimony / cross-examination title 
    colour. The default theme's orange.
    - `3`: thought colour, system message colour. The default theme's
    blue.
    - `4`: robot / text-to-speech machine talk colour. The default
    theme's yellow.

    Notably, the red text colour is missing from this list. That is
    because it is neither used much in the original series, nor can it
    fulfil its purpose of being a "threatening" mod colour due to the
    `"color_custom"` `ICPiece`.
- `"color_custom"`
  - `color: String`: the colour of any following `"text"` `ICPiece`s,
  given in hexadecimal code format.    
  No defaults.    
  If the input given is invalid, the `ICPiece` should be pruned.    
  If no `"text"` `ICPiece` appears after this piece, it should be 
  pruned from the IC message.
- `"delay"`
  - `time: Number`: the client should wait this much in milliseconds
  before continuing to interpret the IC message.    
  No defaults.    
  Cannot be `0` or negative.    
  At least the server should do sanity checks here.
- `"shake"`
  - `intensity: Number`: the intensity with which the screen should
  shake, ranging from `1` to `5`.    
  These values are mostly arbitrary, and their actual strength 
  depends on the client's implementation.    
  No defaults.
- `"bling"`

  No arguments.    
  Equivalent to the *realisation* of pre-2.7.
- `"sfx"`
  - `sound: String`: the name of the sound file that should be played.    
  No defaults.    
  Some other `ICPiece`s, like `"emote"` or `"bling"` can play their
  own sound effects independently.
- `"flip"`
  - `flip: Boolean`: if `true`, the character will appear horizontally 
  flipped.     
  If `false`, it will appear unflipped.    
  Defaults to `false`.
- `"offset"`
  - `offset: Number`: A value from `-100` to `100` that determines how
  far off the user's sprite will appear from its default location, in
  percentages.    
  A value of `-100` is 100% to the left, that is, one whole viewport
  to the left, and a value of `100` is 100% to the right, or one whole
  viewport to the right.    
  Defaults to `0`.
  - `snap: Boolean`: If `true`, the character instantly appears at the
  new offset.     
  If `false`, then the character is translated over there over a short
  amount of time.    
  Defaults to `true`.
- `"shout"`
  - `files: String`: the name of both the interjection animated image 
  and sound file that should happen.    
  No defaults.    
  If the character's folder does not have the sound effect or animated
  image to play, the client can fall back to one set by the theme, or 
  some other default.    
  If the sound effect still cannot be found this way, the game should
  fall back to the interjection whoosh sound effect.    
  If that cannot be found, do not play any sound.    
  If the animated image cannot be found, ignore this piece.    
  The above rule makes this piece the only one that can be "wasted",
  that is, cannot be pruned serverside and not have it count against
  the complexity of the message.

  Furthermore, this is the only `ICPiece` that **can appear as the first `ICPiece` in an IC message** besides `"emote"`, but only if an `"emote"` immediately follows it.    
  In this case, the "camera" is not moved away from the previous
  character on screen, to replicate that "object-into-your-face"
  feeling the original series, and pre-2.7 has with its interjections.
- `"evi"`
  - `id: Number`: the ID of the evidence the user wants to present.
- `"user"`
  - `id: Number`: presents the user profile of a user, and by doing so,
  giving them a notification as well, that they were mentioned.
  Somewhat equivalent of using *callwords* in pre-2.7.

The *absolute minimum* IC message that can be legally sent is a
message consisting of one single `"emote"` `ICPiece`. That is:

```typescript
  {
    header: "IC",
    pieces: [
      {
        header: "emote",
        emote: "normal"
      }
    ]
  }
```

*(Of course, the actual emote called does not matter.)*

The above message is equivalent to sending a single space as the text
in pre-2.7, without any preanimations or special effects, like
realisation or interjections.

Additionally, due to how this system is set up, unlike pre-2.7, it is
possible to have multiple of the same command in one IC message:

```typescript
  {
    header: "IC",
    pieces: [
      {
        header: "emote",
        emote: "handsondesk",
        anim: "slam",
        stall: false
      },
      {
        header: "text",
        text: "Your Honour! "
      },
      {
        header: "delay",
        time: 1500
      },
      {
        header: "text",
        text: "The true culprit of this crime is "
      },
      {
        header: "emote",
        emote: "pointing",
        anim: "point",
        stall: true
      },
      {
        header: "color_theme",
        color: 2
      },
      {
        header: "text",
        text: "this person!"
      },
      {
        header: "evi",
        id: 23
      }
    ]
  }
```

In the above, the user could freely use multiple preanimations and
insert three different kinds of text, and employ various other
roleplay enhancing techniques all in one message.

Further, the server sends back a similarly built packet, however, it
may modify it in some ways.
For example, some servers may filter out delays, or play jokes on the
user's text messages (think disemvowelling and shaking).

[*Back to TOC*][toc]

### OOC messages: `CT` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
  {
    header: "CT",
    message: String
  }
```

*From server:*
```typescript
  {
    header: "CT",
    name: String,
    message: String,
    special: 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8
  }
```

The `name` argument describes the user's name, as given in their
user profile.

The `message` argument is a plain text String detailing what the user
wants to say to other users.
Servers may choose to interpret the messages in any ways (for example,
as command calls).

The `special` argument denotes that the user from whom the message
arrives is notable for some reason.
The client should not be able to add this argument to the packet,
but if it does, it should be ignored.
The valid values for this argument are:
- `0`: user. Nothing special.
- `1`: server. This is a message straight from the server software
itself (feedback, error reporting, etc.).
- `2`: global authority. This user can use some special commands 
across the entire server. Most commonly, these are the mods.
- `3`: local authority. This user can affect only the areas they own.
Most commonly, these are the CMs.
- `4`: casing authority. This is the judge.
- `5`: defence attorney and its cohorts.
- `6`: prosecutor and its cohorts.
- `7`: jurors, for Jury cases.
- `8`: witnesses, for Improv cases.

[*Back to TOC*][toc]

## In-game ##

[*Back to TOC*][toc]

### Change character: `CC` ###

*Client -> Server*    
*Response:* `GUP`

#### Format: ####

```typescript
  {
    header: "CC",
    charid: Number
  }
```

The client requests that the server update its character chosen to the
one with the given character ID in `charid`.

In response, the server should give out updated user profiles, which
contain the characters.

To force specific clients into specific characters, all the server has
to do is give out user profiles that contain the changes.

[*Back to TOC*][toc]

### Music: `MC` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```typescript
  {
    header: "MC",
    name: String,
    fade: Boolean
  }
```

If arriving from the client, it requests that the server play a given
song in the area.
If the packet arrives from the server, it orders the client to play
the song, and loop it.

This packet accepts the following arguments:
- `name`: a String that contains the name of the song, extension
included, that is meant to be played.
- `fade`: if `false`, the music should abruptly switch from one
another (pre-2.7 behaviour). If `true`, the first song should fade
out, then the other should fade in.

Furthermore, requesting that the same song be played as the one
already playing results in stopping the music.
If `fade` was set to `true` when this was done, the music also fades
out into silence.

[*Back to TOC*][toc]

### Change background: `BG` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```typescript
  {
    header: "BG",
    bg: String
  }
```

If sent by the client, requests the server to change background to the
one given by name in `bg`.

If sent by the server, orders the clients to change the background
locally to the one given in `bg`.

[*Back to TOC*][toc]

### Pair up request: `PR` ###

*Client -> Server*    
*Server -> Client*    
*Response:* `PL`

#### Format: ####

```typescript
  {
    header: "PR",
    id: Number
  }
```

If sent by the client, it signals that the sending user wishes to
pair up with the one given by `id`.
As a consequence of this, any previous pairs are broken, even if the
new pair never accepts the request.

If sent by the server, it alerts the target client of a request to
pair up by `id`.

[*Back to TOC*][toc]

### Get list of pairs: `PL` ###

*Server -> Client*

#### Format: ####

```typescript
  {
    header: "PL",
    pairs: [
      {
        id: Number,
        oid: Number
      },
      ...
    ]
  }
```

Returns a list of pairs, where `id` and `oid` are both user IDs.
A pair of `{id: 0, oid: 1}` is equivalent to `{id: 1, oid: 0}`,
that is, pairs are commutative.

Getting a list of pairs should clear the list already existing on
the client.

[*Back to TOC*][toc]

### Check character availability: `CA` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
  {
    header: "CA"
  }
```

*From server:*
```typescript
  {
    header: "CA",
    free: [
      Boolean,
      Boolean,
      Boolean,
      ...
    ]
  }
```

If sent by the client, requests the availability list of characters.

If sent by the server, it updates the client on what characters are
available. 
If `free` at a given index is `true`, then the character with that 
ID is available, else it is false.

[*Back to TOC*][toc]

### Keep alive: `CH` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```typescript
  {
    header: "CH"
  }
```

A simple check that connection is still established.

[*Back to TOC*][toc]

### Login as moderator: `MD` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
  {
    header: "MD",
    pass: String
  }
```

*From server:*
```typescript
  {
    header: "MD",
    result: 0 | 1 | 2
  }
```

If sent by the client, attempt to log in as a moderator using the
given password in `pass`.

In response, the server returns the result of the log-in attempt,
where `result` is:
- `0`: successful login.
- `1`: incorrect password.
- `2`: too many attempts.

[*Back to TOC*][toc]

### Call moderator: `ZZ` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
  {
    header: "ZZ",
    message: String
  }
```

*From server:*
```typescript
  {
    header: "ZZ",
    id: Number,
    area: Number,
    message: String
  }
```

If sent by the client, requests a moderator, with additional reason / 
information if needed.

`message` is optional, defaults to empty text (`""`).

If sent by the server, and the current client recognises that it is
a mod, the user will be alerted, and will be given the caller user's
`id`, the ID of the `area` they called from, and the `message` if they
left any.

[*Back to TOC*][toc]

## Area handling ##

Areas, as a concept, existed for at least 2 years now in *Attorney
Online*.    
The following text merely makes the existence of areas official,
in the sense that they now have their own packets, instead of relying
on the music changing packet to travel between areas, and commands
to get information about areas.

In 2.6, a middle-ground solution was attempted, where the client
filtered out the music list for areas, put them in their own list,
and a special `ARUP` package updated the client on area properties.    
This is not quite the best solution, as it makes far too many
assumptions: that areas exist, that every server would have its areas
in the beginning of the music list, that areas do not have extensions
similar to music (weird, but definitely possible), and, well, that
the server bothered to keep up with updates, and is not running a
custom fork where updating may be harder (or perhaps, as it has
happened before, the custom forks reimplements the feature, but
differently).

[*Back to TOC*][toc]

### Get list of areas: `GA` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
  {
    header: "GA"
  }
```

*From server:*
```typescript
  {
    header: "GA",
    areas: [
      {
        id: Number,
        name: String,
        desc: String,
        players: Number,
        status: String,
        owners: [
          Number,
          Number,
          Number,
          ...
        ],
        protection: 0 | 1 | 2,
        p_area: Number
      }
    ]
  }
```

If sent by the client, requests the list of all areas on the server.

If sent by the server, updates the client on the list of areas.
Every item in the `areas` array describes an area.

An area has the following arguments:
- `id`: a non-negative unique integer, that allows the area to be
referred to.
- `name`: a short, descriptive non-empty title for the area.
- `desc`: a longer text, that describes what the area is about.
- `players`: a non-negative integer that shows the amount of users
currently inside the area.
- `status`: a short text that describes the area's status.    
Some statuses, like `IDLE`, `CASING` and others, could be supported
by the client more than other ones, much like in 2.6, where a few
preset statuses got unique colouration in the area list.
- `owners`: an array of user IDs which tells who are the CMs of a
given area.    
A 'CM' *(Case Maker / Case Manager)* is a user with heightened
privileges in the area they own.
Depending on the area's settings, this allows them to lock the area,
be the sole modifier of evidence, change the area's status, etc.
- `protection`: the level of protection the area currently employs.
These are:
  - `0`: the area is open. Anyone can enter and leave as they please.
  - `1`: spectate only. Anyone who was not inside the area when it
  was made spectate only can join, but can only send OOC messages.    
  Everyone else (those who were inside the area when it was made
  spectate only) can operate as before.    
  Further, those who were inside the area when it was made spectate
  only can also freely leave and come back, and retain their rights.
  (They are considered *invited* to the area.)
  - `2`: the area is locked. Players may enter only if:
    - They were inside the area when it was locked, and left
    (that is, they are still invited).
    - They had been made a CM before the area was locked
    (all CMs are considered invited all the time).
    - They have special privileges (for example, they are mods).
- `p-area`: the ID of the *parent area* of the area, if it exists.
If it does not, this should be `-1`.    
If Area X is a parent area of Area Y, then Area Y is a *child area*
of Area X.
In terms of protection levels, joining areas, etc., child areas are
considered to be "inside" of the parent area.
So if the parent area is locked, everyone inside the parent areas
and its child areas can freely enter and leave the parent area, being
all considered to be inside the parent area when it was locked.    
If a user attempts to join a child area, they must first be eligible
to join its parent area (and its parent area, if it exists, and so
on).    
Note that this does not mean that they are automatically eligible to
join all child areas. If Area X is the parent area, and Areas Y and Z
its children areas, and Z is locked, a user from Y can move through
X as they please, but they cannot enter Z.    
A CM of the parent area is implicitly the CM of all its child areas,
and they cannot be removed from their positions as a CM of the child
areas by other CMs of said areas.
Besides these special restrictions, a parent area and its child areas
are considered distinct -- so a user from a parent area will not
hear messages from its child areas, or vice versa.

[*Back to TOC*][toc]

### Become an owner of an area: `CM` ###

*Client -> Server*    
*Server -> Client*    
*Response:* `GA`

#### Format: ####

```typescript
  {
    header: "CM",
    id: Number
  }
```

If sent by the server, makes the given user with `id` an owner of the
current area they are in.

A user cannot become the owner of an area they are not present in
at the time of the promotion to owner.

If sent by the client, it serves two distinct purposes:
- The client can send its own user ID, trying to claim the area,
if possible.    
An area can be claimed (by default) if it has no owners, or if one of
the already existing owners CM-ed the user (see below).
- The client can send some other user's ID, allowing them to promote
themselves to a co-owner, if they so wish.    
As stated above, the other user must be in the area the current user
is for this to work.

When a user is promoted to CM, a `GA` packet should be sent, to update
the clients' knowledge of the area.

[*Back to TOC*][toc]

### Give up ownership of an area: `UM` ###

*Client -> Server*    
*Server -> Client*    
*Response:* `GA`

#### Format: ####

```typescript
  {
    header: "CM",
    id: Number
  }
```

Forces a user with the given `id` to lose ownership of an area.

Much like with the `CM` packet, a user can give its own user ID to
willingly give up ownership of an area,
or give some other user's ID to force them to step down.

Additionally, a mod can force anyone to step down from the owner
position.

Sending this packet to the user who was demoted is purely for
aesthetic functions (if the client adds buttons or such in regards
to areas the user owns).    
Like with the `CM` packet, a `GA` packet should be sent to update
knowledge about the area.

[*Back to TOC*][toc]

### Edit area: `EA` ###

*Client -> Server*    
*Response:* `GA`

#### Format: ####

```typescript
  {
    header: "EA",
    name: String,
    desc: String,
    status: String,
    protection: 0 | 1 | 2
  }
```

Edits some of the basic features of an area.

It is up to the server to determine who can edit what areas.    
Generally, it should be only allowed to CMs, and the extent of the
edit is also dependent on the server.    
For example, in pre-2.7, only CMs could lock or unlock areas, but
anyone could edit the status, choosing between pre-set statuses.

For explanations about the various arguments, read up on the `GA`
packet.

A `GA` packet should be sent if the edit is accepted to update
knowledge about the area.

[*Back to TOC*][toc]

### Join area: `JA` ###

*Client -> Server*    
*Response:* `MA`

#### Format: ####

```typescript
  {
    header: "JA",
    id: Number
  }
```

Sends a request to the server that the current user wishes to join
the area with the ID `id`.

[*Back to TOC*][toc]

### Move to area: `MA` ###

*Server -> Client*

#### Format: ####

```typescript
  {
    header: "MA",
    id: Number,
    success: 0 | 1 | 2 | 3
  }
```

Gives the client a status update about how its user moved to area
`id`.

The `success` arguments can be as follows:
- `0`: successfully moved by the user's will.
- `1`: the user was forcibly moved to the area.
- `2`: the user successfully moved to the area, but has no rights to
speak in it (because it is spectate only, for example).
- `3`: the user could not move to the area, because it was locked.

Do note that this merely gives a *status update* to the client:
the actual moving should be done on the server's side.

[*Back to TOC*][toc]

### Invite to area: `IA` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
  {
    header: "IA",
    id: Number
  }
```

*From server:*
```typescript
  {
    header: "IA",
    id: Number,
    area: Number
  }
```

If sent by the client, sends out an invite to the user `id` to the
area the client's user resides in.

If sent by the server, notifies the target client's user of them being
invited to the area with the ID `area` by the user `id`.

It is up to the server to manage who can invite and when.

Being invited to an area means that the user there can access all
functions besides OOC messages (within their rights) if it is set to
spectate only, or join and leave the area even if it is locked.

An invitation is only lost if the user gets specifically uninvited,
or if they disconnect from the server.

[*Back to TOC*][toc]

### Uninvite from area: `UA` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```typescript
  {
    header: "UA",
    id: Number
  }
```

*From server:*
```typescript
  {
    header: "UA",
    area: Number
  }
```

If sent by the client, uninvites the user `id`.
See `IA` and `GA` for explanations of inviting and uninviting.

If sent by the server, notifies the user that they are no longer
considered invited in area `area`.    
Do note that in this case, the uninviter's ID is not stated.

[*Back to TOC*][toc]

[ao2protocol]: https://github.com/AttorneyOnline/AO2Protocol/blob/master/Attorney%20Online%20Client-Server%20Network%20Specification.md
[ao3protocol]: https://github.com/AttorneyOnline/AO3Protocol/blob/master/animated-chatroom-design.md
[ao2.7assets]: https://gist.github.com/oldmud0/bdda25ebfc1ab4c68bb38b21585327f9
[toc]: #user-content-table-of-contents