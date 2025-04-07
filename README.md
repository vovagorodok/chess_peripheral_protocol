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
- [Variant](#variant)
  - [Chess 960](#chess-960)
- [Feature](#feature)
  - [Message](#message)
  - [Last move](#last-move)
  - [Round](#round)
  - [Score](#score)
  - [Time](#time)
  - [Side](#side)
  - [History](#history)
  - [Prefen](#prefen)
  - [Option](#option)
- [Contributors](#contributors)
- [Links](#links)

## Assumptions
When central and peripheral has the same pieces positions we call it synchronized.  
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
Required commands:
```
c) begin <fen>
p) sync <fen>
p) unsync <fen>
p) state <fen>
cp) move <uci>
c) ok
c) nok
c) promote <uci>
c) end <reason>
```
`ok` and `nok` commands are used for acknowelage, which must be answer for some commands

Functionality can be extended by commands:
```
c) variant <variant>
c) feature <feature>
```

Example:
```
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) sync rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
c) move a2a3
p) move a7a6
c) ok
```

## Begin
Required commands: `begin`, `sync`, `unsync`.  
Central always begins round by `begin`, peripheral responces by `sync` or `unsync`.  
Peripheral send `sync` if has the same state:
```
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) sync rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
```
Peripheral send `unsync` if has different state:
```
c) begin rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) unsync rnbqkbnr/pppppppp/8/8/8/7P/1PPPPPPP/RNBQKBNR b
```
Peripheral can send even `w` (white), `b` (black), `?` (unknown) instead of full piece information depending on internal sensors and knowelage:
```
p) unsync ????????/????????/8/8/8/8/????????/????????
```
```
p) unsync bbbbbbbb/bbbbbbbb/8/8/8/8/wwwwwwww/wwwwwwww w
```

## Unsync
Required commands: `sync`, `unsync`, `state`.  
During the round, peripheral can detect and indicate boards mismatch by sending `unsync`.  
As example cat knocked down all white pieces (cat disaster case):
```
p) unsync ????????/????????/8/8/8/8/8/8
```
Peripheral send `unsync` if has different state, then peripheral can send `state` until both states become the same:
```
p) unsync rnbqkbnr/pppppppp/8/8/8/7P/1PPPPPPP/RNBQKBNR
p) state rnbqkbnr/pppppppp/8/8/8/8/1PPPPPPP/RNBQKBNR
p) sync rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
```

## Move
Required commands: `move`, `ok`, `nok`.  
Only central controls round and should accept or reject peripheral moves:
```
p) move a2a3
c) ok
p) move a7a6
c) nok
```
Peripheral should not send responces for central moves:
```
c) move a7a4
```
Castling is indicated as king move:
```
c) move e1g1
```
En passant is indicated as pawn diagonal move:
```
c) move a5b6
```
Central and peripheral send moves with promotion:
```
p) move a7a8q
```

## Promote
Required commands: `promote`.  
Cantral will promote instead peripheral by sending `promote` instead `ok` or `nok` if peripheral can't distinquish promotion:
```
p) move a7a8
c) promote a7a8n
```

## End
Required commands: `end`.  
Peripheral should not response:
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

## Variant
Required commands: `variant`, `set_variant`.  
Cenral can check all supported variants:
```
c) variant standard
p) ok
c) variant chess_960
p) nok
c) variant 3_check
p) ok
```

Sending `set_variant` means switch to them if supported.  
Central should sent `variant` only before `begin`:
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
Required commands: `feature`.  
Cenral can check all features that are supported by peripheral.  
If central didn't ask for feature, then it should be disabled on both sides:
```
c) feature last_move
p) ok
c) feature check
p) nok
c) feature msg
p) ok
```

### Message
Feature name is `msg`.
```
c) feature msg
```

Provides command:
```
cp) msg <message>
```

Used to show message in another side.
```
c) msg Hello peripheral
p) ok
```
```
p) msg Hello central
c) ok
```

### Last move
Feature name is `last_move`.
```
c) feature last_move
```

Provides command:
```
c) last_move <uci>
```

Can be send only after central `fen`.  
When round doesn't have last move, then command shouldn't be sent.
```
c) fen rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR w
p) ok
c) last_move a2a3
p) ok
```

### Round
Feature name is `round`.
```
c) feature round
```

Provides commands:
```
cp) draw
cp) resign
c) check <king position>
c) checkmate <king position>
c) stalemate <king position>
c) end
```

Central or peripheral can offer draw. If opposite side accept then round ends with draw result.
```
c) draw
p) ok
```

If opposite side reject then round continues.
```
p) draw
c) nok
```

Central or peripheral can indicate round resignation that can't be rejected.
```
p) resign
c) ok
```

Central can indicate round check, checkmate and stalemate that can't be rejected.
```
c) check a3
p) ok
```

Central can indicate round ending in case of other varints with other ending rules.
```
c) end
p) ok
```

### Score
Feature name is `score`.
```
c) feature score
```

Provides command:
```
c) score <white> <black>
```

Central can indicate score after round ending.
```
c) score 1.5 2.5
p) ok
```

### Time
Feature name is `time`.
```
c) feature time
```

Provides command:
```
c) time <white> <black>
```

Central can indicate remaining time of each side in miliseconds.
```
c) time 31444 12510
p) ok
```

### Side
Feature name is `side`.
```
c) feature side
```

Provides command:
```
c) side <color>
```

Central can indicate peripheral side where color can be `w` (white), `b` (black), `?` (both).
```
c) side w
p) ok
```

### History
Feature name is `history`.
```
c) feature history
```

Provides commands:
```
cp) undo <uci>
cp) rendo <uci>
```

Central or peripheral can offer move round state. If opposite side accept then round state changes.
```
c) undo a3a2
p) ok
```

If opposite side reject then round continues with the same sate.
```
p) rendo a2a3
c) nok
```

### Prefen
Feature name is `prefen`.
```
c) feature prefen
```

Provides commands:
```
cp) prefens_begin
cp) prefens_end
cp) prefen <fen>
```
Can be used to check opposite state.  
For example central can check and begin round from peripheral state.  
Central or peripheral can request `prefen` broadcasting each time when opposite state changes.
```
c) prefens_begin
p) ok
p) prefen rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR w
c) ok
p) prefen rnbqkbnr/pppppppp/8/8/P7/8/1PPPPPPP/RNBQKBNR w
c) ok
c) prefens_end
p) ok
```

### Option
Feature name is `option`.
```
c) feature option
```

Provides commands:
```
c) options_begin
p) option_item <name> <type> <value> <type params>
p) options_end
cp) option <name> <value>
```

Central can read all options from peripheral.
```
c) options_begin
p) ok
p) option_item brightness int 4 1 4 1
c) ok
p) option_item animation_speed int 2 1 8 1
c) ok
p) options_end
c) ok
```

Central or peripheral can change option.
```
c) option brightness 3
p) ok
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
p) option_item option_name bool false
```
Enum contains predefined values.
```
p) option_item option_name enum value2 value1 value2 value3
```
Str contains any text information.
```
p) option_item option_name str Hello world
```
Int step param is optional.
```
p) option_item option_name int 0 0 8
```
Float step param is optional.
```
p) option_item option_name float 0.2 -0.1 0.5
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