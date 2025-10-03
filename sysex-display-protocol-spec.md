# MIDI SYSEX Display Protocol Specification
Version 1.0.0-draft.3

## Overview
This document specifies the MIDI System Exclusive (SYSEX) message format for controlling a vector-based display device. The protocol supports basic drawing primitives (lines, rectangles, circles), text rendering, bitmap icons, and interactive MIDI control elements. Commands can be combined and sequenced to create complex display updates, with support for chunking large updates across multiple messages.

## Table of Contents
- [Command Summary](#command-summary)
- [Display Specifications](#display-specifications)
- [SYSEX Message Format](#sysex-message-format)
  - [Header Structure](#header-structure)
  - [Command Chunking](#command-chunking)
  - [Parameter Encoding](#parameter-encoding)
  - [Command Delimiter](#command-delimiter)
- [Drawing Commands](#drawing-commands)
  - [Line Drawing (0x13)](#line-drawing-0x13)
  - [Text Drawing (0x14)](#text-drawing-0x14)
  - [Rectangle Drawing (0x15)](#rectangle-drawing-0x15)
  - [Circle Drawing (0x17)](#circle-drawing-0x17)
  - [Icon Drawing (0x16)](#icon-drawing-0x16)
  - [Control Drawing (0x18)](#control-drawing-0x18)
- [Implementation Notes](#implementation-notes)

## Command Summary
| Command | ID | Bytes | Description |
|---------|:--:|:----:|-------------|
| [Line](#line-drawing-0x13) | 0x13 | 12 | Draws a line between two points |
| [Text](#text-drawing-0x14) | 0x14 | 7+n | Renders text using built-in fonts |
| [Rectangle](#rectangle-drawing-0x15) | 0x15 | 13 | Draws filled or outlined rectangle |
| [Icon](#icon-drawing-0x16) | 0x16 | 8 | Draws a predefined bitmap icon |
| [Circle](#circle-drawing-0x17) | 0x17 | 10 | Draws filled or outlined circle |
| [Control](#control-drawing-0x18) | 0x18 | 11+n | Draws interactive MIDI control element |

All commands are terminated with delimiter byte 0x7F.

## Display Specifications
- Display Resolution: Up to 4095x4095 pixels (12-bit addressing)
- Color Values: 0-126 [0x00-0x7E]

## SYSEX Message Format

### Header Structure
Every SYSEX message starts with:
```
F0 [id1] [id2] [id3] [proto1] [proto2] [version] [command_type] [chunk_count] [seq_number] [command_data...] F7
```

Where:
- `F0`: Start of SYSEX
- `[id1] [id2] [id3]`: 3-byte Manufacturer ID assigned by MMA/AMEI
- `[proto1] [proto2]`: 2-byte Device protocol version (vendor-specific)
- `version`: Protocol specification version, must be 0x00 for this version
- `command_type`: 0x03 for Vector Display commands
- `chunk_count`: Total number of chunks in the complete sequence 1-127 [0x01-0x7F]
- `seq_number`: Sequence number of this chunk 0-127 [0x00-0x7F]
- `command_data`: One or more drawing commands, each terminated by 0x7F delimiter
- `F7`: End of SYSEX

### Command Chunking
The protocol supports two levels of chunking:

1. Message Chunking: Large display updates can be split across multiple SYSEX messages using sequence numbers
2. Command Chunking: Multiple drawing commands can be combined within a single message

#### Message Chunking
When display updates are too large for a single SYSEX message, they can be split into multiple chunks:
- `chunk_count`: Total number of messages in the sequence 1-127 [0x01-0x7F]
- `seq_number`: Position of this message in the sequence (0 to chunk_count-1, 0x00-0x7E)
- The receiver should buffer and process chunks in sequence
- Complete display update is rendered when all chunks are received

#### Command Chunking
Multiple drawing commands can be combined into a single SYSEX message to optimize transmission and rendering. Each command within the chunk must be terminated with the command delimiter (0x7F). Commands in a chunk are processed sequentially in the order they appear.

Chunk format:
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 [command_type] [chunk_count] [seq_number] [command1...] 7F [command2...] 7F [...] F7
```

Example - Draw a rectangle with text inside (single chunk):
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 01 00  # Single chunk (count=1, seq=0)
   15 00 0A 00 0A 00 32 00 14 01 01 00 00 7F    # Rectangle at (10,10), width=50, height=20, filled
   14 00 0F 00 0F 01 00 4D 45 4E 55 7F           # Text "MENU" at (15,15)
F7
```

Example - Complex display update split across two messages:
```
# First chunk (count=2, seq=0)
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 02 00
   15 00 0A 00 0A 00 64 00 32 01 01 00 00 7F    # Large rectangle
   14 00 0F 00 0F 01 00 54 49 54 4C 45 7F        # Title text
F7

# Second chunk (count=2, seq=1)
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 02 01
   18 00 01 02 00 01 00 7F 4F 4B 7F              # OK button
   16 00 32 00 32 01 01 00 7F                    # Icon
F7
```

Example - Create a button with an icon:
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03
   18 02 00 01 00 01 00 00 53 45 54 7F           # Button labeled "SET" at grid (0,1)
   16 00 01 00 01 02 01 00 7F                    # Icon index 2 next to button
F7
```

When chunking commands:
- Commands are rendered in order, allowing layering effects
- Each command must be properly terminated with 0x7F
- Total message length (including all chunks) must not exceed MIDI SYSEX size limits
- Invalid commands in a chunk are ignored while valid commands are still processed

### Parameter Encoding
Coordinate and size parameters (x, y, width, height, diameter) use 12-bit encoding split across two bytes. Each byte can contain values 0-0x3F (0-63), with 0x7F reserved as a command delimiter:
```
First byte:  [00] [######]  (low 6 bits)
Second byte: [00] [######]  (high 6 bits)
Value = (byte2 << 6) | byte1  (0-4095) where byte1=low 6 bits, byte2=high 6 bits
```
This provides a range of 0-4095 which is then clamped to the valid range for each parameter type. Note that the high 2 bits of each byte must be 0.

### Command Delimiter
Each drawing command in the message must be terminated with the command delimiter byte 0x7F. This byte cannot appear within command parameters. Multiple commands can be sent in a single SYSEX message, each terminated by the delimiter.

### Drawing Commands

#### Line Drawing (0x13)
Draws a line between two points.
```
13 [x1_low] [x1_high] [y1_low] [y1_high] [x2_low] [x2_high] [y2_low] [y2_high] [color] [thickness] [line_style] 7F
```
- x1, y1: Start coordinates (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- x2, y2: End coordinates (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- color: Color value 0-126 [0x00-0x7E]
- thickness: Line thickness in pixels 1-126 [0x01-0x7E]
- line_style: Line style (0=solid, 1=dotted1, 2=dotted2, 3=dotted3)

Example:
Draw a line from (0,32) to (255,32), thickness 2, solid style:
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 13 00 00 20 00 3F 03 20 00 01 02 00 7F F7
```

#### Text Drawing (0x14)
Renders text using built-in bitmap fonts.
```
14 [x_low] [x_high] [y_low] [y_high] [color] [size] [text...] 7F
```
- x, y: Top-left coordinates (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- color: Color value 0-126 [0x00-0x7E]
- size: Scale factor (0=normal, 1=2x, 2=3x)
- text: ASCII characters

Example:
Draw "Hello" at (10,10), normal size:
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 14 0A 00 0A 00 01 00 48 65 6C 6C 6F 7F F7
```

#### Rectangle Drawing (0x15)
Draws a filled or outlined rectangle.
```
15 [x_low] [x_high] [y_low] [y_high] [width_low] [width_high] [height_low] [height_high] [color] [line_width] [filled] [line_style] 7F
```
- x, y: Top-left coordinates (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- width: Rectangle width (12-bit encoded as low 6 bits then high 6 bits, range 1-4095)
- height: Rectangle height (12-bit encoded as low 6 bits then high 6 bits, range 1-4095)
- color: Color value 0-126 [0x00-0x7E]
- line_width: Outline thickness in pixels 1-126 [0x01-0x7E] (default 1)
- filled: 0=outline only, 1=filled
- line_style: Line style (0=solid, 1=dotted1, 2=dotted2, 3=dotted3)

Example:
Draw a filled rectangle at (10,10) with width=50, height=20, color=1:
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 15 0A 00 0A 00 32 00 14 00 01 01 01 00 7F F7
```

#### Circle Drawing (0x17)
Draws a filled or outlined circle.
```
17 [x_low] [x_high] [y_low] [y_high] [diameter_low] [diameter_high] [color] [line_width] [filled] 7F
```
- x, y: Top-left coordinates (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- diameter: Circle diameter (12-bit encoded as low 6 bits then high 6 bits, range 1-4095)
- color: Color value 0-126 [0x00-0x7E]
- line_width: Outline thickness in pixels 1-126 [0x01-0x7E] (default 1)
- filled: 0=outline only, 1=filled

Example:
Draw a filled circle with diameter 20 at (30,20):
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 17 1E 00 14 00 14 00 01 01 01 7F F7
```

#### Icon Drawing (0x16)
Draws a predefined bitmap icon. Icons are bitmap patterns stored in the device's memory. The available icons and their corresponding indices are device-specific. If an invalid icon index is specified, the behavior is implementation-defined.
```
16 [x_low] [x_high] [y_low] [y_high] [icon_index] [color] [size] 7F
```
- x, y: Top-left coordinates (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- color: Color value 0-126 [0x00-0x7E]
- size: Scale factor (0=normal, 1=2x, 2=3x)

Example:
Draw an icon at (20,30), normal size:
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 16 14 00 1E 00 00 01 00 7F F7
```

#### Control Drawing (0x18)
Draws an interactive MIDI control element with dynamic value display.
```
18 [type] [x_low] [x_high] [y_low] [y_high] [style] [color] [value_low] [value_high] [label...] 7F
```
- type: Control type (0=fader, 1=knob, 2=button, 3=encoder)
- x, y: Top-left coordinates (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- style: Visual style and orientation
  - For faders: 0=vertical, 1=horizontal
  - For knobs/encoders: 0=dot indicator, 1=line indicator, 2=fill indicator
  - For buttons: 0=momentary, 1=toggle
- color: Color value 0-126 [0x00-0x7E]
- value: Current value (12-bit encoded as low 6 bits then high 6 bits, range 0-4095)
- label: ASCII encoded control label, terminated by command delimiter (0x7F)

Example:
Draw a vertical fader labeled "Volume" at coordinates (100,50) with value at 50%, color 1:
```
F0 [id1] [id2] [id3] [proto1] [proto2] 00 03 18 00 24 01 32 00 00 01 3E 1F 56 6F 6C 75 6D 65 7F F7
```
Breakdown:
- type = 0 (fader)
- x = 100 (encoded as low=0x24, high=0x01)
- y = 50 (encoded as low=0x32, high=0x00)
- style = 0 (vertical)
- color = 1
- value = 2046 (about 50%, encoded as low=0x3E, high=0x1F)
- label = "Volume" in ASCII (56 6F 6C 75 6D 65)

## Implementation Notes

1. Multiple commands can be sent either as separate SYSEX messages or combined in chunks within a single message
2. Commands are rendered in the order received, with later commands drawing over earlier ones
3. Invalid commands or malformed messages should be ignored
4. When processing chunked commands:
   - Each chunk must be properly terminated with the command delimiter (0x7F)
   - The entire message must be properly framed with F0...F7
   - Individual invalid commands in a chunk are skipped while valid ones are processed
   - Consider MIDI buffer size limitations when chunking commands
   - For multi-message updates:
     - Buffer incoming chunks until complete sequence is received
     - Verify chunk_count and seq_number for consistency
     - Process commands in sequence order
     - Discard incomplete sequences after timeout