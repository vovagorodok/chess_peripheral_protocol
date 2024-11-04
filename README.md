# Chess Peripheral Protocol
String-based protocol that opens the possibility to connect and play chess from an app (which we call 'central') with devices like electronic boards, clocks, and others (called 'peripherals') in a common way.

## Table of contents
- [Physical layer](#physical-layer)
  - [Bluetooth LE implementation](#bluetooth-le-implementation)
- [Assumptions](#assumptions)
- [Basic functionality](#basic-functionality)
- [Fen](#fen)
- [Move](#move)
- [Promote](#promote)
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

## Physical Layer
The protocol should be universal enough to allow implementations through different physical interfaces like RS-232, BLE, and more.

### Bluetooth LE implementation
Each peripheral should advertise on service with two string characteristics that simulate serial interface.  
Following uuids are defined:
````
service: f5351050-b2c9-11ec-a0c0-b3bc53b08d33
tx characteristic: f53513ca-b2c9-11ec-a0c1-639b8957db99
rx characteristic: f535147e-b2c9-11ec-a0c2-8bbd706ec4e6
````

You can use our arduino implementation as reference:
https://github.com/vovagorodok/ArduinoBleChess

## Assumptions
All commands should be written in lower case where words are splitted by the `_` sign.  
Central always controls round rules.

Designation of device that sends command:  
`c)` - central  
`p)` - peripheral  
`cp)` - central or peripheral  

## Basic functionality
Required commands:
```
cp) ok
cp) nok 
cp) fen <fen>
cp) move <uci>
c) promote <uci>
```
`ok` and `nok` commands are used for acknowelage, which must be answer for resto of command besiede `promote`

Functionality can be extended by commands:
```
c) variant <variant>
c) feature <feature>
```

Example:
```
c) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) ok
c) move a2a3
p) ok
p) move a7a6
c) ok
```

## Fen
Command name is `fen`.  
Rapresents round state.
Central always starts round by sending `fen`.
Peripheral send `ok` if has the same state.
```
c) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) ok
```
Peripheral send `nok` if has different state, then central or peripheral can send `fen` until both state become the same.
```
c) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) nok
p) fen rnbqkbnr/pppppppp/8/8/8/7P/1PPPPPPP/RNBQKBNR
c) nok
p) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
c) ok
```
Peripheral can send even `w` (white), `b` (black), `?` (unknown) instead of full piece information depending on internal sensors and knowelage.  
```
p) fen ????????/????????/8/8/8/8/????????/????????
```
```
p) fen bbbbbbbb/bbbbbbbb/8/8/8/8/wwwwwwww/wwwwwwww w
```
During the round, peripheral can detect and indicate boards mismatch by sending `fen`. As example cat knocked down all white pieces (cat disaster case).
```
c) move a2a3
p) ok
p) fen ????????/????????/8/8/8/8/8/8
c) nok
```

## Move
Command name is `move`.  
Central and peripheral can send moves only when states are same (central and peripherial are synchronised).
```
c) move a2a3
p) ok
p) move a7a6
c) ok
```
Only central can reject moves.
```
p) move a7a4
c) nok
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
p) move a7a8q
c) ok
```

## Promote
Command name is `promote`. 
Cantral will promote instead peripheral if peripheral can't distinquish promotion.
```
p) move a7a8
c) promote a7a8n
p) ok
```

## Variant
Command name is `variant`.  
Represents round variant.
Cenral can check all supported variants.
```
c) variant standard
p) ok
c) variant chess_960
p) nok
c) variant 3_check
p) ok
```

Sending `variant` means switch to them if supported.  
Central should sent `variant` only before `fen`.  
If `variant` has not been sent then `standard` should be used.
```
c) variant standard
p) ok
c) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) ok
```

If peripheral doesn't support variant then central shouldn't send `fen` for them.
```
c) variant chess_960
p) nok
```

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
Variant name is `chess_960`.  
Known also as fischer random.  
Same as `standard`, but castling is indicated as king and rook starting positions.
```
c) move e1h1
```

## Feature
Command name is `feature`.  
Cenral can check all features that are supported by peripheral.  
If central didn't ask for feature, then it should be disabled on both sides.
```
c) feature last_move
p) ok
c) feature time
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
c) score [white] [black]
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
c) time [white] [black]
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
cp) prefen [fen]
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
string
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
String contains any text information.
```
p) option_item option_name string Hello world
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
