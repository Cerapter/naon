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
  - [Downloading assets](#user-content-downloading-assets)
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

## Legend ##

- **`AO` / `AO2`:** Attorney Online 2
- **`NAON`:** new AO netcode -- a shorthand for this very suggestion.
- **`...`:** the argument list continues with the pattern established
before it.

[*Back to TOC*](#user-content-table-of-contents)

## Packet setup ##

By default, every packet should begin with a *header:* a short string
that identifies it.

[*Back to TOC*](#user-content-table-of-contents)

## Handshake ##

[*Back to TOC*](#user-content-table-of-contents)

### Establishing connection: `HI` ###

*Client -> Server*    
*Response:* `SD`

#### Format: ####

```json
  {
    "header": "HI",
    "client": String,
    "ver": String,
    "vver": String
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

[*Back to TOC*](#user-content-table-of-contents)

### Send server details: `SD` ###

*Server -> Client*

#### Format: ####

```json
  {
    "header": "SD",
    "name": String,
    "desc": String,
    "players": int,
    "maxplayers": int,
    "protection": int
  }
```

Sent in response to a `HI` packet. The server explains some basic
details about itself.

Most arguments are self-explanator, however, the `protection` argument
can take three different values:
- `0`: open. Anyone can join.
- `1`: password or spectate. An user can only join as a player if they
give the correct password. 
However, they can choose to spectate if they do not know it.
- `2`: password needed. The only way to join the server, player or
spectator, is if the user knows the password.

[*Back to TOC*](#user-content-table-of-contents)

### Join server: `JS` ###

*Client -> Server*    
*Response:* `JR`

#### Format: ####

```json
  {
    "header": "JS",
    "spectate": bool,
    "password": String
  }
```

The client sends a request to join the server.

If `spectate` is true, the client wants to join as a spectator.
Else, it wants to join as a player.

The `password` string contains the client's guess at the server's
password, assuming it is passworded.

[*Back to TOC*](#user-content-table-of-contents)

### Server join result: `JR` ###

*Server -> Client*

#### Format: ####

```json
  {
    "header": "JR",
    "result": int,
    "message": String
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

[*Back to TOC*](#user-content-table-of-contents)

### Disconnect: `DC` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```json
{
  "header": "DC",
  "message": String
}
```

*From server:*
```json
{
  "header": "DC",
  "id": int,
  "message": String
}
```

If sent by the client, formally closes the connection to the server,
while giving a reason for leaving in the `message` argument.

If sent by the server, it announces the disconnection of the client
with the given ID, and their reason to leave in `message`.

In both cases, `message` is optional, and will default to `"User
disconnected"` if not specified or left empty.

To actually force people to quit, see `KK` and `KB`.

[*Back to TOC*](#user-content-table-of-contents)

### Kick: `KK` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```json
{
  "header": "KK",
  "id": int,
  "message": String
}
```

*From server:*
```json
{
  "header": "KK",
  "message": String
}
```

If sent by the client, requests that the server kick the player with
the given ID, and the reason in `message`.

If sent by the server, kicks the client it is sent to.
It should force the client to disconnect, though the server should
make sure to cease connection as well, to avoid trickery.

In both cases, `message` is option, and defaults to `""`.
Keeping the default value does not display a reason.

[*Back to TOC*](#user-content-table-of-contents)

### Ban: `KB` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```json
{
  "header": "KB",
  "id": int,
  "message": String
}
```

*From server:*
```json
{
  "header": "KB",
  "message": String
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

[*Back to TOC*](#user-content-table-of-contents)

## Loading ##

[*Back to TOC*](#user-content-table-of-contents)

### Downloading assets ###

*To be decided. I'm still not sure how we plan on handling assets.*

[*Back to TOC*](#user-content-table-of-contents)

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

[*Back to TOC*](#user-content-table-of-contents)

### Set user profile: `SUP` ###

*Client -> Server*    
*Response:* `GUP`

#### Format: ####

```json
  {
    "header": "SUP",
    "name": String,
    "desc": String
  }
```

Sends the user's preferred OOC name and a short description of them.

Both arguments are optional. If they are left out, or left empty 
(`""`), the server should assign the user a unique (server-wide) name,
and the description "???".

[*Back to TOC*](#user-content-table-of-contents)

### Get user profiles: `GUP` ###

*Server -> Client*

#### Format: ####

```json
  {
    "header": "GUP",
    "users": [
      {
        "id": int,
        "name": String,
        "desc": String,
        "charid": int
      },
      ...
    ]
  }
```

Gives the client a list of user profiles from the server.    
Can be used for both an area-wide user profile list, or a server-wide 
user profile list.

If a client gets a `GUP` packet, it should delete all previously
stored user profiles.

An `User Profile` consists of:
- `id`: the user's ID on the server.
- `name`: the user's OOC name.
- `desc`: the user's description.
- `charid`: the ID of the character the user is currently playing as.

[*Back to TOC*](#user-content-table-of-contents)

## Messaging ##

[*Back to TOC*](#user-content-table-of-contents)

### IC messages: `IC` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```json
  {
    "header": "IC",
    "pieces": [
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
  "missigno", and be considered immediately finished, having no actual
  animation.
  - `stall: bool`: if `true`, the client should not continue
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
  - `center: bool`: if `true`, the message will appear centered.    
  If another `"text"` `ICPiece` appears later on, that has a `center`
  with the opposite value, that one is ignored.    
  The first `"text"` piece determines whether the entire text is
  centered or not.
- `"speed"`
  - `speed: int`: the speed at which any following `"text"` `ICPiece`
  should be displayed, in percentages.    
  Defaults to `100`.    
  Larger values imply faster speeds.    
  Sanity checks are heavily encouraged here, both on client and server
  side.    
  A value of `0` should be rejected outright.    
  If no `"text"` `ICPiece` comes after this piece, it should be pruned
  from the IC message.
- `"color_theme"`
  - `color: int`: the colour of any following `"text"` `ICPiece`s,
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
    fulfill its purpose of being a "threatening" mod colour due to the
    `"color_custom"` `ICPiece`.
- `"color_custom"`
  - `color: String`: the colour of any following `"text"` `ICPiece`s,
  given in hexadecimal code format.    
  No defaults.    
  If the input given is invalid, the `ICPiece` should be pruned.    
  If no `"text"` `ICPiece` appears after this piece, it should be 
  pruned from the IC message.
- `"delay"`
  - `time: int`: the client should wait this much in miliseconds 
  before continuing to interpet the IC message.    
  No defaults.    
  Cannot be `0` or negative.    
  At least the server should do sanity checks here.
- `"shake"`
  - `intensity: int`: the intensity with which the screen should
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
  - `flip: bool`: if `true`, the character will appear horizontally 
  flipped.     
  If `false`, it will appear unflipped.    
  Defaults to `false`.
- `"offset"`
  - `offset: int`: A value from `-100` to `100` that determines how
  far off the user's sprite will appear from its default location, in
  percentages.    
  A value of `-100` is 100% to the left, that is, one whole viewport
  to the left, and a value of `100` is 100% to the right, or one whole
  viewport to the right.    
  Defaults to `0`.
  - `snap: bool`: If `true`, the character instantly appears at the
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
  - `id: int`: the ID of the evidence the user wants to present.
- `"user"`
  - `id: int`: presents the user profile of an user, and by doing so,
  giving them a notification as well, that they were mentioned.
  Somewhat equivalent of using *callwords* in pre-2.7.

The *absolute minimum* IC message that can be legally sent is a
message consisting of one single `"emote"` `ICPiece`. That is:

```json
  {
    "header": "IC",
    "pieces": [
      {
        "header": "emote",
        "emote": "normal"
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

```json
  {
    "header": "IC",
    "pieces": [
      {
        "header": "emote",
        "emote": "handsondesk",
        "anim": "slam",
        "stall": false
      },
      {
        "header": "text",
        "text": "Your Honour! "
      },
      {
        "header": "delay",
        "time": 1500
      },
      {
        "header": "text",
        "text": "The true culprit of this crime is "
      },
      {
        "header": "emote",
        "emote": "pointing",
        "anim": "point",
        "stall": true
      },
      {
        "header": "color_theme",
        "color": 2
      },
      {
        "header": "text",
        "text": "this person!"
      },
      {
        "header": "evi",
        "id": 23
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
user's text messages (think disemvoweling and shaking).

[*Back to TOC*](#user-content-table-of-contents)

### OOC messages: `CT` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```json
  {
    "header": "CT",
    "message": String
  }
```

*From server:*
```json
  {
    "header": "CT",
    "name": String,
    "message": String,
    "special": int
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

[*Back to TOC*](#user-content-table-of-contents)

## In-game ##

[*Back to TOC*](#user-content-table-of-contents)

### Change character: `CC` ###

*Client -> Server*    
*Response:* `GUP`

#### Format: ####

```json
  {
    "header": "CC",
    "charid": int
  }
```

The client requests that the server update its character chosen to the
one with the given character ID in `charid`.

In response, the server should give out updated user profiles, which
contain the characters.

To force specific clients into specific characters, all the server has
to do is give out user profiles that contain the changes.

[*Back to TOC*](#user-content-table-of-contents)

### Music: `MC` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```json
  {
    "header": "MC",
    "name": String,
    "fade": bool
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

[*Back to TOC*](#user-content-table-of-contents)

### Change background: `BG` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```json
  {
    "header": "BG",
    "bg": String
  }
```

If sent by the client, requests the server to change background to the
one given by name in `bg`.

If sent by the server, orders the clients to change the background
locally to the one given in `bg`.

[*Back to TOC*](#user-content-table-of-contents)

### Pair up request: `PR` ###

*Client -> Server*    
*Server -> Client*    
*Response:* `PL`

#### Format: ####

```json
  {
    "header": "PR",
    "id": int
  }
```

If sent by the client, it signals that the sending user wishes to
pair up with the one given by `id`.
As a consequence of this, any previous pairs are broken, even if the
new pair never accepts the request.

If sent by the server, it alerts the target client of a request to
pair up by `id`.

[*Back to TOC*](#user-content-table-of-contents)

### Get list of pairs: `PL` ###

*Server -> Client*

#### Format: ####

```json
  {
    "header": "PL",
    "pairs": [
      {
        "id": int,
        "oid": int
      },
      ...
    ]
  }
```

Returns a list of pairs, where `id` and `oid` are both user IDs.
A pair of `{"id": 0, "oid": 1}` is equivalent to `{"id": 1, "oid": 0}`,
that is, pairs are commutative.

Getting a list of pairs should clear the list already existing on
the client.

[*Back to TOC*](#user-content-table-of-contents)

### Check character availability: `CA` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```json
  {
    "header": "CA"
  }
```

*From server:*
```json
  {
    "header": "CA",
    "free": [
      bool,
      bool,
      bool,
      ...
    ]
  }
```

If sent by the client, requests the availability list of characters.

If sent by the server, it updates the client on what characters are
available. 
If `free` at a given index is `true`, then the character with that 
ID is available, else it is false.

[*Back to TOC*](#user-content-table-of-contents)

### Keep alive: `CH` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

```json
  {
    "header": "CH"
  }
```

A simple check that connection is still established.

[*Back to TOC*](#user-content-table-of-contents)

### Login as moderator: `MD` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```json
  {
    "header": "MD",
    "pass": String
  }
```

*From server:*
```json
  {
    "header": "MD",
    "result": int
  }
```

If sent by the client, attempt to log in as a moderator using the
given password in `pass`.

In response, the server returns the result of the log-in attempt,
where `result` is:
- `0`: successful login.
- `1`: incorrect password.
- `2`: too many attempts.

[*Back to TOC*](#user-content-table-of-contents)

### Call moderator: `ZZ` ###

*Client -> Server*    
*Server -> Client*

#### Format: ####

*From client:*
```json
  {
    "header": "ZZ",
    "message": String
  }
```

*From server:*
```json
  {
    "header": "ZZ",
    "id": int,
    "area": int,
    "message": String
  }
```

If sent by the client, requests a moderator, with additional reason / 
information if needed.

`message` is optional, defaults to empty text (`""`).

If sent by the server, and the current client recognises that it is
a mod, the user will be alerted, and will be given the caller user's
`id`, the ID of the `area` they called from, and the `message` if they
left any.

[*Back to TOC*](#user-content-table-of-contents)

[ao2protocol]: https://github.com/AttorneyOnline/AO2Protocol/blob/master/Attorney%20Online%20Client-Server%20Network%20Specification.md
[ao3protocol]: https://github.com/AttorneyOnline/AO3Protocol/blob/master/animated-chatroom-design.md
[messagepack]: https://msgpack.org/