# AR-REG1U Model Fujitsu Mini-split Remote Infrared Signals

This document outlines how the AR-REG1U Fujitsu Mini-split IR remote encodes signals to send to the wall-mounted mini-split unit. It's entirely possible other Fujitsu remotes, or even other mini-split manufacture's remotes, operate on the same principles. However, the AR-REG1U is the remote I have and the following was derived by capturing signals from the remote using custom software and reverse engineering the captured signals.

## Signal Encoding

The AR-REG1U encodes binary data by varying the duration the IR transmitter in the remote is OFF. The wall-mounted mini-split unit receives these pulsating signals (the ON/OFF of the remote's IR transmitter), converts them to binary, and then decodes them. As you might expect, before you press a button on the remote, the transmitter is OFF. Once you press a button, say to increase the desired temperature, the remote's IR transmitter starts flashing. _Every_ command starts with the same duration pulse, likely to act as a signal for the mini-split unit to pay attention a signal is coming:

**ON** for ~3.25 ms  
**OFF** for ~1.60 ms  

After this initial pulse, the rest of the signal represent binary. Each pulse ON is about 0.4ms, however, as mentioned earlier the significant portion of the signal is when the IR transmitter is OFF in between ON pulses. If the OFF duration between pulses is ~0.4ms, this is a binary 0. If the OFF duration is closer to 1.2ms, this is a binary 1. Using this logic we can encode arbitrary binary values in the same format used by the AR-REG1U remote. For instance, let's encode the number 1234 or 10011010010 in binary. Read the following table left to right, top to bottom. All times are in milliseconds.

| IR LED ON (ms) | IR LED OFF (ms) | BINARY VALUE |
|--|--|-- |
| 3.25 | 1.6 | N/A |
| 0.4 | 1.2 | 1 |
| 0.4 | 0.4 | 0 |
| 0.4 | 0.4 | 0 |
| 0.4 | 1.2 | 1 |
| 0.4 | 1.2 | 1 |
| 0.4 | 0.4 | 0 |
| 0.4 | 1.2 | 1 |
| 0.4 | 0.4 | 0 |
| 0.4 | 0.4 | 0 |
| 0.4 | 1.2 | 1 |
| 0.4 | 0.4 | 0 |

It may be surprising that the meaningful portion of the signal is when the LED is OFF, but consider that the LED draws no power when it is OFF. It is likely slightly more efficient to vary the duration on the OFF portion of the cycle while leaving the ON portion as short as possible.

Here is an example signal captured via an oscilliscope. Note that the IR receiver proped to capture this signal produces a HIGH when there is no IR signal present and a LOW when there is (so inverted from what you might expect).

![](waveform.png?raw=true)

Note the long LED ON (LOW) and LED OFF (HIGH) pulse at the start followed by the standard binary representation. As you'll see below, the binary data following this initial pulse aligns with the command header (0x28C...).

## Command Representation
There are two broad types of commands the remote sends: those that encode temperature, mode, and fan selections, and those that do not (which I will refer to as _Action Commands_).

### Temperature, Mode, and Fan Commands
These commands are 128 bits in length. They encode the following data:

 - Temperature: An integer representing the temperature above 16°C. *
 - Mode of Operation: AUTO, COOL, DRY, FAN, HEAT
 - Fan Speed: AUTO, HUGH, MEDIUM, LOW, QUIET
 - Fan Mode: STATIC, SWING
 - Command: SET_TEMPERATURE, TURN_DEVICE_ON

\* The remote converts to Fahrenheit by taking the temperature offset, multiplying it by 2, and adding that value to 60°F.

Much of this information is split into 4-bit chunks (conveniently 1 hex character), referred to as nibbles below. Consider the following:
```
28C60008087F900CXXX0XX00000004XX
|______________|
    HEADER
    
The first 8 bytes (first 16 nibbles) are a header that is stable for all commands.
Nibble 17 encodes whether to turn the device on or only set temperature if the device is already on.
Nibble 18 encodes the temperature offset.
Nibble 19 encodes the mode of operation.
Nibble 21 encodes the fan speed.
Nibble 22 encodes the fan mode.
The last byte (nibbles 31 and 32) are a checksum.
```
When reading/writing the nibbles, it is important to note that the AR-REG1U reverses the order of bits within a single nibble. To clarify, the decimal number 4 (0100) would be represented in the bit stream as 0010. This is the case for all 5 nibbles mentioned above as well as the checksum.

#### Mode of Operation
```
AUTO = 0
COOL = 1
DRY  = 2
FAN  = 3
HEAT = 4
```

#### Fan Speed
```
AUTO   = 0
HIGH   = 1
MEDIUM = 2
LOW    = 3
QUIET  = 4
```

#### Fan Mode
```
STATIC = 0
SWING  = 1
```

#### Command
```
SET_TEMPERATURE = 0
TURN_DEVICE_ON  = 1
```

Therefore, an IR command to set a minisplit (without the checksum) that is already on into `COOL` mode with `AUTO` fan speed at `20°C (68°F)` in `STATIC` fan mode would be:
```
Nibble 17: SET_TEMPERATURE = 0 = 0000 = 0000 = 0
Nibble 18: 20 - 16         = 4 = 0100 = 0010 = 2
Nibble 19: COOL            = 1 = 0001 = 1000 = 8
Nibble 21: AUTO            = 0 = 0000 = 0000 = 0
Nibble 22: STATIC          = 0 = 0000 = 0000 = 0
28C60008087F900C02800000000004XX
```

#### Checksum
The last byte of the signal is the checksum. This is important due to the unreliable nature of IR transmission. A checksum is used so that the minisplit system can be confident that it received a valid and complete signal. The checksum is calculated by adding together bytes 9 through 14 (after reversing the bits in them) inclusive, then subtracting that sum from, what I refer to as, the _checksum root_ and, finally, bit-reversing the result. For the AR-REG1U, the _checksum root_ is 176.

Consider the following psuedo code:
```
    private static byte GenerateChecksum(Signal signal, int checksumRoot)
    {
        int sum = 0;
        for (int index = 8; index < 14; index++)
        {
            sum += signal.GetReversedByte(index);
        }
        return Signal.ReverseByte((byte)((checksumRoot - sum) % 256));
    }
```

Using the example signal from before, we would calculate the checksum as follows:
```
28C60008087F900C02800000000004XX

9:  02 = 0000 0010 = 0100 0000 = 64
10: 80 = 1000 0000 = 0000 0001 = 1
11: 00 = 0000 0000 = 0000 0000 = 0
12: 00 = 0000 0000 = 0000 0000 = 0
13: 00 = 0000 0000 = 0000 0000 = 0
14: 00 = 0000 0000 = 0000 0000 = 0
---------------------------------------
+                                65

176 - 65 = 111 = 0110 1111 = 1111 0110 = F6

28C60008087F900C02800000000004F6
```

### Action Commands
These commands are 56 bits in length. I have not been as successful divining the meaning of the individual bits in these commands; however, given that they perform a single action without any configuration, a look-up table is as good as constructing the commands yourself.

```
28C6000808XXXX
|________|
  HEADER
  
The first 5 bytes are a header that is stable for all commands.
The last 2 bytes differ for each command in ways I do not yet understand.
```

Turn device OFF: `28C600080840BF`  
Change fan angle (SET button): `28C600080836C9`  
Toggle economy mode: `28C6000808906F`  
Toggle powerful mode: `28C60008089C63`

## Producing Signals

Using the information above, it should be trivial to produce a signal that would have the outcome you want. The only thing that is important to remeber is the initial pulse before binary data starts.
