# Chess Peripheral Protocol
String-based protocol that opens the possibility to connect and play chess from an app (which we call `central`) with devices like electronic boards, clocks, and others (called `peripheral`) in a common way.

## Table of contents
- [Assumptions](#assumptions)
- [Physical layer](#physical-layer)
  - [Bluetooth LE](#bluetooth-le)
- [Basic functionality](#basic-functionality)
- [Begin](#begin)
- [Unsync](#unsync)
- [Move](#move)
- [Promote](#promote)
- [End](#end)
- [Error](#error)
- [Variant](#variant)
  - [Chess 960](#chess-960)
- [Feature](#feature)
  - [Message](#message)
  - [Undo](#undo)
  - [Last Move](#last-move)
  - [Check](#check)
  - [Moved](#moved)
  - [Resign](#resign)
  - [Draw Offer](#draw-offer)
  - [Draw Reason](#draw-reason)
  - [Variant Reason](#variant-reason)
  - [Side](#side)
  - [Score](#score)
  - [Time](#time)
  - [State Stream](#state-stream)
  - [Get State](#get-state)
  - [Set State](#set-state)
  - [Option](#option)
- [Contributors](#contributors)
- [Links](#links)

## Assumptions
When central and peripheral has the same state (pieces positions) we call it synchronized.  
Always central controls round rules and peripheral controls synchronization.  
All commands and parameters should be written in lower case where words are splitted by the `_` sign.

Designation of device that sends command:  
`c)` - central  
`p)` - peripheral  
`cp)` - central or peripheral 

## Physical Layer
The protocol should be universal enough to allow implementations through different physical interfaces like USB, BLE, and more.

### Bluetooth LE
Each peripheral should advertise on service with two string characteristics that simulate serial interface.  
Following uuids are defined:
````
service: f5351050-b2c9-11ec-a0c0-b3bc53b08d33
tx characteristic: f53513ca-b2c9-11ec-a0c1-639b8957db99
rx characteristic: f535147e-b2c9-11ec-a0c2-8bbd706ec4e6
```` 

## Basic functionality
Require commands:
```
c) begin <fen>
p) sync <fen>
p) unsync <fen>
p) state <fen>
cp) move <uci>
c) promote <uci>
c) end <reason>
cp) ok
cp) nok
cp) err <message>
```
Functionality can be extended by commands:
```
c) feature <feature>
c) variant <variant>
c) set_variant <variant>
```
Example:
```
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) sync rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
c) move a2a3
p) move a7a6
c) ok
c) end resign
```

## Begin
Require commands:
```
c) begin <fen>
p) sync <fen>
p) unsync <fen>
```
Central always begins round by `begin`, peripheral responses by `sync` or `unsync`.  
Peripheral send `sync` if has the same state.
```
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) sync rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
```
Peripheral send `unsync` if has different state.
```
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) unsync rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR b
```
Peripheral can send even `w` (white), `b` (black), `?` (unknown) instead of full piece information depending on internal sensors and knowledge.
```
p) unsync ????????/????????/8/8/8/8/????????/????????
```
```
p) unsync bbbbbbbb/bbbbbbbb/8/8/8/8/wwwwwwww/wwwwwwww w
```

## Unsync
Require commands:
```
p) sync <fen>
p) unsync <fen>
p) state <fen>
```
Peripheral can send `state` until both states become the same.
```
p) unsync rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR
p) state rnbqkbnr/pppppppp/8/8/8/8/1PPPPPPP/RNBQKBNR
p) sync rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
```
During the round, peripheral can detect and indicate boards mismatch by sending `unsync`.  
As example cat knocked down all white pieces (cat disaster case).
```
p) unsync ????????/????????/8/8/8/8/8/8
```

## Move
Require commands:
```
cp) move <uci>
c) ok
c) nok
```
Only central controls round and should accept or reject peripheral moves.
```
p) move a2a3
c) ok
p) move a7a6
c) nok
```
Peripheral should not send responses for central moves.
```
c) move a7a4
```
Castling is indicated as king move.
```
c) move e1g1
```
En passant is indicated as pawn diagonal move.
```
c) move a5b6
```
Central and peripheral send moves with promotion.
```
c) move a7a8q
```
Central can send moves even when unsynchronized.
```
p) unsync bbbbbbbb/bbbbbbbb/8/8/8/8/wwwwwwww/wwwwwwww w
c) move a7a8
```

## Promote
Require commands:
```
c) promote <uci>
```
Central will promote instead peripheral by sending `promote` instead `ok` or `nok` if peripheral can't distinguish promotion.
```
p) move a7a8
c) promote a7a8n
```

## End
Require commands:
```
c) end <reason>
```
Peripheral should not response.
```
c) end checkmate
```

List of predefined end reasons:
```
undefined
checkmate
stalemate
draw
timeout
resign
abort
```

## Error
Require commands:
```
cp) err <message>
```
Used to show error in another side.
```
p) err Unsupported command
```

## Variant
Require commands:  
```
c) variant <variant>
c) set_variant <variant>
p) ok
p) nok
```
Central can check all supported variants.
```
c) variant standard
p) ok
c) variant chess_960
p) nok
c) variant 3_check
p) ok
```

Sending `set_variant` means switch to them if supported.  
Central should sent `set_variant` only before `begin`.
```
c) set_variant standard
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
```
If `set_variant` was never send then `standard` should be used.  
If peripheral doesn't support variant then central shouldn't send `begin` for them.

List of predefined variants names:
```
standard
chess_960
3_check
atomic
king_of_the_hill
anti_chess
horde
racing_kings
crazy_house
```
If specific variant doesn't have chapter it means that it use exactly the same commands as `standard`.

### Chess 960
Variant name: `chess_960`.  
Known also as fischer random.  
Same as `standard`, but castling is indicated as king and rook starting positions.
```
c) move e1h1
```

## Feature
Require commands:
```
c) feature <feature>
p) ok
p) nok
```
Central can check all features that are supported by peripheral.  
If central didn't ask for feature, then it should be disabled on both sides.
```
c) feature last_move
p) ok
c) feature check
p) nok
c) feature msg
p) ok
```

### Message
Feature `msg` require commands:
```
cp) msg <message>
```
Used to show message in another side.
```
c) msg Hello peripheral
```
```
p) msg Hello central
```

### Undo
Feature `undo` require commands:
```
cp) undo <uci>
c) ok
c) nok
```
Only central controls round and should accept or reject peripheral undo moves as for `move`.
```
p) undo a2a3
c) ok
p) undo a7a6
c) nok
c) undo a7a4
```
Castling, En passant and promotion are indicated as for `move` command.
```
c) undo a7a8q
```
```
p) undo a7a8
c) promote a7a8q
```

### Last Move
Feature `last_move` require commands:
```
c) last_move <uci>
```
When round doesn't have last move, then command shouldn't be sent.  
Can be send only after: `begin`, `undo`.
```
c) begin rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR w
c) last_move a2a3
```
```
p) undo a7a6
c) ok
c) last_move a2a3
```

### Check
Feature `check` require commands:
```
c) check <king position>
```
When round doesn't have check, then command shouldn't be sent.  
Can be send only before `end` and after: `begin`, `last_move`, `move`, `undo`.
```
c) begin rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR w
c) check a2
```
```
p) move a7a6
c) ok
c) check a2
c) end checkmate
```

### Moved
Feature `moved` require commands:
```
p) moved
```
Can be used mostly for mechanical devices where `move` and `undo` take more time.
```
c) move a2a3
p) moved
c) undo a2a3
p) moved
```

### Resign
Feature `resign` require commands:
```
p) resign
```
Peripheral can indicate round resignation that can't be rejected.
```
p) resign
```

### Draw Offer
Feature `draw_offer` require commands:
```
cp) draw_offer
cp) ok
cp) nok
```
If opposite side accept then round ends with draw result. Central should not send `end` command.
```
c) draw_offer
p) ok
```
If opposite side reject then round continues.
```
p) draw_offer
c) nok
```

### Draw Reason
Feature `draw_reason` require commands:
```
c) end <reason>
```
Draw reasons can be:
- Draw Offer – Mutual Agreement, both players agree to a draw.
- Threefold Repetition – The same position occurs three times with the same possible moves (claimed by a player).
- Fifty-Move Rule – If 50 moves pass without a pawn move or capture, a draw can be claimed.
- Insufficient Material – Neither player has enough pieces to checkmate (e.g., king vs. king, king and knight vs. king).
- Dead Position – The position is such that no sequence of legal moves can lead to checkmate (e.g., blocked pawns with no way to progress).

Central can send draw reason instead draw.
```
c) end fifty_move
```

List of predefined end reasons:
```
draw_offer
threefold_repetition
fifty_move
insufficient_material
dead_position
```

### Variant Reason
Feature `variant_reason` require commands:
```
c) end <reason>
```
Central can send variant reason instead undefined.
```
c) end king_of_the_hill
```
List of predefined end reasons:
```
3_check
king_of_the_hill
```

### Side
Feature `side` require commands:
```
c) side <color>
```
Central can indicate peripheral side where color can be `w` (white), `b` (black), `?` (both).
```
c) side w
```
Central should sent `side` only before `begin`.
```
c) set_variant standard
c) side ?
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
```

### Score
Feature `score` require commands:
```
c) score <white> <black>
```
Central can indicate score after round ending in format `x.0` or `x.5` if draw.
```
c) score 1.5 2.5
```

### Time
Feature `time` require commands:
```
c) time <white> <black>
```
Central can indicate remaining time of each side in milliseconds.
```
c) time 31444 12510
```
Can be send only after: `begin`, `move`, `last_move`, `undo`.

### State Stream
Feature `state_stream` require commands:
```
cp) state <fen>
```
Central or peripheral indicate `state` each time when state changes even in synchronized state in order to indicate pieces removing and adding before move.
```
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
p) sync rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
p) state rnbqkbnr/pppppppp/8/8/8/8/1PPPPPPP/RNBQKBNR
p) state rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR
p) move a2a3
c) ok
p) state rnbqkbnr/1ppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR
p) state rnbqkbnr/1ppppppp/p7/8/8/P7/1PPPPPPP/RNBQKBNR
c) move a7a6
```

### Get State
Feature `get_state` require commands:
```
c) get_state
p) state <fen>
```
Central can check and begin round from peripheral state.
```
c) get_state
p) state rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
```
Peripheral can send `state` for each change until `begin`.

### Set State
Feature `set_state` require commands:
```
c) set_state
p) unsync_setible <fen>
```
Central can request autocomplete if peripheral can do it by indicating `unsync_setible` instead `unsync`.
```
c) begin rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR b
p) unsync_setible rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
c) set_state
p) sync rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR b
```

### Option
Feature `option` require commands:
```
c) options_begin
p) option <name> <type> <value> <type params>
p) options_end
c) options_reset
cp) set_option <name> <value>
```
Central can read all options from peripheral.
```
c) options_begin
p) option brightness int 4 1 4 1
p) option animation_speed int 2 1 8 1
p) options_end
```

Central or peripheral can change option.
```
c) set_option brightness 3
```

Option types and their params:
```
bool
enum <values>
str
int <min> <max> <optional step>
float <min> <max> <optional step>
```

Bool has only `true` and `false` values.
```
p) option option_name bool false
```
Enum contains predefined values.
```
p) option option_name enum value2 value1 value2 value3
```
Str contains any text information.
```
p) option option_name str Hello world
```
Int step param is optional.
```
p) option option_name int 0 0 8
```
Float step param is optional.
```
p) option option_name float 0.2 -0.1 0.5
```

Predefined option names that can be reused:
```
brightness
engine_level
move_inactive_time
move_speed
animation_speed
animation_type
always_queen_promotion
enable_smooth_animation
enable_docks
enable_beeps
enable_shakes
```

## Contributors

## Links
CECP:  
http://hgm.nubati.net/CECP.html   
https://www.gnu.org/software/xboard/engine-intf.html  
UCI:  
http://wbec-ridderkerk.nl/html/UCIProtocol.html  
Libraries:  
https://github.com/vovagorodok/ArduinoBleChess