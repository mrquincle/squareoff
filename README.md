# Square off

The square off is an automated chess board. The app communicates over Bluetooth towards the board.

It very often takes a long time before it connects, this bothered me a lot. Hence, I delved into the bluetooth communication.

## Bluetooth

There's a nice tool for reading/writing from Bluetooth devices from Nordic Semi, <https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-mobile>.

On a Linux machines use something like this to find out the MAC address (but the Nordic app can scan itself as well).

    sudo hcitool lescan --duplicates | grep Square

When you connect to the chess board (set RSSI threshold low and have your phone close to the board), you will see that there is a Nordic UART Characteristic.

There's an RX characteristic (which is to **write**). There's a TX characteristic (which is to **read**). Yes, the opposite of what you would expect. The RX/TX terminology is from the perspective of the chess board.

Register to the notifications on the TX characteristic.

## Protocol

Each command is a string and is prefixed by an x and postfixed by a z.

Start

    xCONNECTEDz
    xBOARDTYPEz

This probably gives back the board type towards the phone, probably including some versioning. For me it is GKS, 36v5 and 8.

Start the game with the AI as black

    xGAMEBLACKz

Start the AI part (this command is given from the phone, the chess board itself is dumb)

    xd2d4z

It responds with `12-OK*`.

Move on the board yourself by pressing the piece at the start and end position. You will now see a notification of the move you played. Indicate that you have received it.

    xOKz

To get battery level

    xBATVOLy

It will return something like

    6-11.63*

Or, with the adapter charging at the wall socket

    2-85.*

Note that the `.` can be anything non-ascii, here it is `0x0a = 10`. And actually ending with `0x0d = 14, 0x0a = 10` after the asterisk, `0x2a = 42`.


# Streaming

The instructions above is with 1 party sending moves over Bluetooth, the other playing on the board.

To just follow the moves of a game we need to send all moves to the board. Rather than `xGAMEBLACKz`, start streaming with

    xLIVEz

# Ending

To end the game is something like

    xRSTVARz

The board doesn't care if it won.

# Set back the pieces

Setting the pieces back is a bit more convoluted. Keep tight.

    xAUTORESETz

Is something like

    x5|2|3|2|7|5|3|2|z
    ...
    xSIZEDONEz
    x0,-0.08|1,0|2.08z
    x,0|5,7|5,2|2.92,z
    ...
    xCODONEz

The first string is a collection of the number of moves that have to be done with the same piece.
So, the first piece in the above array moves 5 times.

Then, we concatenate everything together. The above lines:

    (0,-0.08) -> (1,0) -> (2.08,0) -> (5,7) -> (5,2)

The decimal values are tweaks to the motors (I think to reduce overshoot or something like that), just round them up

    (0,0) -> (1,0) -> (2,0) -> (5,7) -> (5,2)

We start with the first 0 as being column `a` and the second to be row `1` (just one more because it is here zero-indexed). So, it becomes:

    a1 -> b1 -> c1 -> f8 -> f3

This sequence does not make sense, but you get the drift.

    x2,2|1.5,1.5|1.5,z
    x0.5|0.92,-0.08|3z

This is a movement by the knight on c3 (2,2) diagonally to the point in between the fields c3 and b2 (1.5,1.5) then to the point in between the fields c2 and b1 (1.5,0.5) and then to its base place b1 (1,0) or here slight adjusted to (0.92-0.08).

The columns next to the board are numbered `-2`, `-1` next to column `a` and `8`, `9` next to column `h`.

Finally everything ends with an okay coming back from the board

    15-OK*

# Todo

There's not much to be figured out by now.

* How to start with white? That is probably xGAMEWHITEz
* How to indicate promotion to Queen, Rock, Bishop, Knight?
* How to indicate castling?
* How to indicate an en passant move?

