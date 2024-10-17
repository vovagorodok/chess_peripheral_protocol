# Chess Peripheral Protocol
This protocol opens posibility to connect and play chess with peripheral devices like electronic boards, clocks and others in a common way.

## Table of contents
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
  - [Option](#option)

## Bluetooth LE implementation
Each peripheral should advertise on service with two string characteristics that simulate serial interface.  
Following uuids are defined:
````
service: f5351050-b2c9-11ec-a0c0-b3bc53b08d33
tx characteristic: f53513ca-b2c9-11ec-a0c1-639b8957db99
rx characteristic: f535147e-b2c9-11ec-a0c2-8bbd706ec4e6
````

## Assumptions
All commands should be written in lower case where words are splitted by the `_` sign.  
Round rules should be always controlled by central.

Designation of device that sends command:  
`c)` - central  
`p)` - peripheral  
`cp)` - central or peripheral  
It should not be visible in real communication.

## Basic functionality
Required commands:  
`cp) ok/nok` - command response  
`cp) fen [fen]` - round state  
`cp) move [uci]` - piece move  
`c) promote [uci]` - piece promotion

Functionality can be extended by commands:  
`c) variant [variant]` - round variant  
`c) feature [feature]` - feature request

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
Central always starts round by sending `fen`. Peripheral send `ok` if has the same state.
```
c) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) ok
```
Peripheral send `nok` if has different state. Than central or peripheral send `fen` until both state become the same.
```
c) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) nok
...
p) fen rnbqkbnr/pppppppp/8/8/8/7P/1PPPPPPP/RNBQKBNR
c) nok
...
p) fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
c) ok
```
Peripheral can send even `w` (white), `b` (black), `?` (unknown) instead of full piece information depending on internal sensors and knovelage.  
```
...
p) fen ????????/????????/8/8/8/8/????????/????????
c) nok
...
p) fen bbbbbbbb/bbbbbbbb/8/8/8/8/wwwwwwww/wwwwwwww w
c) ok
```
During the round, peripheral can detect and indicate boards mismatch (cat disaster case) by sending `fen`. As example cat knocked down all white pieces.
```
...
c) move a2a3
p) ok
p) fen ????????/????????/8/8/8/8/8/8
c) nok
```

## Move
Central and peripheral send moves only when states math.
```
c) move a2a3
p) ok
p) move a7a6
c) ok
```
Only central can reject moves.
```
...
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

## Promote
Central and peripheral send moves with promotion.
```
p) move a7a8q
c) ok
```

Cantral will promote instead peripheral if peripheral is not smart enough.
```
p) move a7a8
c) promote a7a8n
p) ok
```

## Variant
Cenral can check all supported variants.
```
c) variant standard
p) ok
c) variant chess960
p) nok
c) variant 3check
p) ok
```

Sending `variant` means switch to them if supported.
Central should sent `variant` command bifore `fen`.
If `variant` is not sent than `standard` should be used.
```
c) variant standard
p) ok
c) frn rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
p) ok
```
Peripheral can not support variant. Than central should not send `fen`.
```
c) variant chess960
p) nok
```

List of predefined variants names.
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

### Chess 960
Same as `standard`, but castling is indicated as king and rook starting positions.
```
c) move e1h1
```

## Feature
Cenral can check all supported features.
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
p) ok
```
Show message in other side.
```
c) msg Hello peripheral
p) ok
...
p) msg Hello central
c) ok
```

### Last move
Feature name is `last_move`.
```
c) feature last_move
p) ok
```
Provides `last_move` command that should be send only after `fen`.  
When round doesn't have last move, command shouldn't be sent.
```
c) frn rnbqkbnr/pppppppp/8/8/8/P7/1PPPPPPP/RNBQKBNR w
p) ok
c) last_move a2a3
p) ok
```

### Option
Feature name is `option`.
```
c) feature option
p) ok
```
```
c) options_begin
p) ok
p) option leds_brightness 4 range 1 4 1
c)
...
p) options_end
c) ok
...
c) option leds_brightness 3 # set option
p) ok
...
p) option leds_brightness 2 # inform device option changed
c) ok
```


## Links
CECP:  
http://hgm.nubati.net/CECP.html   
https://www.gnu.org/software/xboard/engine-intf.html  
UCI:  
http://wbec-ridderkerk.nl/html/UCIProtocol.html
