; -*- Mode: Markdown; -*-
; vim: filetype=none tw=76 expandtab

# ErlyMUD

ErlyMUD is a rather minimalistic MUD server, written in Erland and making
use of the excellent OTP libraries. The aim is to have solid support for
exploration and roleplaying, within a highly fault-tolerant environment
where system crashes or reboots are more of an exotic curiosity than a
commonplace thing. 

Erlang/OTP is an excellent match for MUD development, with strong support
for concurrency through light-weight processes, hot code upgrades, near-
transparent mechanisms for distributed computing, etc.

This project started out as a way for me to learn Erlang/OTP, within a 
context that I knew something about. Since those early days three weeks 
ago, it's actually developed into something marginally usable. We'll see
where this goes.

## Current Status

ErlyMUD presents what's currently at least a minimally playable environment.
It's possible to connect, create a password-protected user account, and log
in to the game. Once there, you can communicate with other players, walk
around between rooms, and handle items.

## Game Features

    * Rooms have a title, and brief + long descriptions.
        * The brief description is used when walking into / through a room, and is 
          intended to only show the most obvious features of the room. 
        * When using the "look" command, the long description will be shown instead.
    * Items can be picked up, dropped and looked at;
        * "get sword", "drop sword"
        * If an item is 'attached', it can't be picked up. Instead it belongs to
          the room, and is used to add detail descriptions so that you can for
          example do "look painting" and see a more in-depth description of that
          part of the room.
    * You can see who is logged on, talk to other players, and use emotes.
        * "who", "tell <who> <what>", "say <something>", "emote <something>"
    * Navigation is currently restricted to the basics;
        * "go west", "west", etc..

## Technical Stuff

Processes are heavily used in order to obtain concurrency and fault isolation.

When someone connects, they initially get a Connection process, which just
handles gen_tcp abstraction and telnet option negotiation, immediately passing
input along to a Session process. The Session has a stack of Request Handlers,
in the form of {M, F, A} tuples. When a line of input is received, the Session
initiates a Request process, passing along the MFA tuple for the current 
Request Handler. When the input has been acted upon, the Request process goes 
away and the Session awaits the next input line.

During login, the User and Living processes get created, representing the user
account and the in-game incarnation, respectively. The Connection, Session,
User and Living processes are all linked - which currently means that if one
dies, so will the rest. This is to avoid process leakage in case of errors. 

In the future, the intent is to intelligently handle process failures, so that 
if for example the Living process dies, the User process can just start a new
one, allowing the player to go on as if nothing happened (mostly). Likewise,
if the User process dies, the Session should kick you back to the login prompt
(without disconnecting the socket), and when you've enter username and pass-
word, you should get reconnected to the existing Living process.

This kind of error handling is already done for the Game and Room processes,
which keep player info in their state. The Game process will trap exit signals
from the User process, and remove the user from its list of logged-on users.
The Room process will trap exit signals from the Living process and remove it
from its inventory of people in the room.

Room loading happens completely dynamically. When a player logs on, the game
tries to load the room they were in by calling em_room_mgr:get_room(RoomName)
which will first see if the room is already loaded, and if it's not then it
will simply load the room.

To avoid problems, em_room_mgr:get_room/1 will always verify that a Room Pid
is alive before returning it, since a Room might have crashed. If it's not
alive, then again it'll simply be loaded and replaced in the ETS table.

## Getting Started

Currently, things aren't really set up in a way so that's it's easy for an
Erlang newbie to get things up and running. This will hopefully improve.

However, try the following steps to start ErlyMUD directly from the
development environment, on a Linux system:

    0. Make sure you have Erlang R14B01 installed, to avoid having to edit
       the ErlyMUD .rel file
    1. Download the latest snapshot from https://bitbucket.org/jwarlander/erlymud/downloads,
       currently that means erlymud-0.3.0.(zip|tar.gz|tar.bz2)
    2. Unpack the source code somewhere, you'll get an "erlymud" directory
    3. Go to this directory and compile the source:
        $ cd erlymud
        $ ./rebar compile
    4. Start an Erlang shell, with the ErlyMUD ebin path added:
        $ erl -pa lib/erlymud/ebin/
    5. Create a local boot script, then quit the Erlang shell:
        1> systools:make_script("erlymud-0.3.0", [local]).
        2> q().
    6. Start up ErlyMUD:
        $ erl -boot ./erlymud-0.3.0
    7. From another terminal, connect to the game and create a user:
        $ telnet localhost 1155
    8. Have fun!

That's all really. Better documentation might happen sooner or later,
if this project doesn't die off ;)

If you have any questions, send an e-mail to johan@snowflake.nu and I'll
try to help you.

Johan Wärlander
