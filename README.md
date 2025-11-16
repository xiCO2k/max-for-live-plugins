# Max for Live Plugin Creation Guide

This guide documents the process for creating Max for Live (.amxd) MIDI plugins from scratch or by modifying existing ones.

## Key Insights

### File Format
- `.amxd` files are **binary files**, not plain text or directories
- They use a specific binary format with headers: `ampf`, `meta`, `ptch`
- The structure is: **Binary Header + JSON Patcher Data**
- File type shows as "data" when checked with `file` command

### Binary Structure
```
Offset  | Content
--------|--------------------------------------------------
0x00    | "ampf" (4 bytes)
0x04    | 0x04 0x00 0x00 0x00 (4 bytes)
0x08    | "mmmmm" (5 bytes)
0x0D    | "meta" (4 bytes)
0x11    | 0x04 0x00 0x00 0x00 (4 bytes)
0x15    | 0x00 0x00 0x00 0x00 (4 bytes)
0x19    | "ptch" (4 bytes)
0x1D    | Length of JSON (4 bytes, big-endian)
0x21    | JSON patcher data (variable length)
```

## Method 1: Copy and Modify Existing Device

This is the **RECOMMENDED** approach as it preserves the correct binary format.

### Steps:

1. **Find a working template device** (preferably one that does something similar)
   ```bash
   # Example: Using an existing MIDI converter
   cp "/path/to/existing.amxd" "/path/to/new_device.amxd"
   ```

2. **Modify the device using Python**
   ```python
   import struct

   # Read the working file as binary
   with open("source.amxd", "rb") as f:
       data = f.read()

   # Find where JSON starts (after binary header)
   header_end = data.find(b'{')
   header = data[:header_end]
   json_and_tail = data[header_end:]

   # Convert JSON to string using latin-1 encoding
   json_str = json_and_tail.decode('latin-1')

   # Make text replacements in the JSON
   json_str = json_str.replace('"text" : "oldObject"', '"text" : "newObject"')

   # Convert back to bytes
   new_json = json_str.encode('latin-1')

   # Update the length header (bytes 20-24, big-endian)
   new_header = bytearray(header)
   struct.pack_into('>I', new_header, 20, len(new_json))

   # Write new file
   with open("new_device.amxd", "wb") as f:
       f.write(new_header)
       f.write(new_json)
   ```

3. **Create symlink to Ableton's MIDI Effects folder**
   ```bash
   ln -s "/path/to/your/device.amxd" \
         "/Users/USERNAME/Desktop/ABLETON LIVE SOUNDS/Library/User Library/Presets/MIDI Effects/Max MIDI Effect/YourDevice.amxd"
   ```

## Common Max Objects for MIDI Processing

### Input Objects
- `midiin` - Receives all MIDI input
- `touchin` - Receives aftertouch messages
- `ctlin N` - Receives CC number N (e.g., `ctlin 11` for CC11)
- `notein` - Receives note messages
- `bendin` - Receives pitch bend

### Output Objects
- `midiout` - Outputs MIDI messages
- `touchout` - Outputs aftertouch messages
- `ctlout N` - Outputs CC number N (e.g., `ctlout 11` for CC11)
- `noteout` - Outputs note messages
- `bendout` - Outputs pitch bend
- `pgmout` - Outputs program change
- `polyout` - Outputs polyphonic aftertouch

### Processing Objects
- `midiparse` - Parses MIDI into separate streams
- `midiselect @touch` - Filters only aftertouch messages
- `midiselect @ctl N` - Filters only CC number N
- `route` - Routes messages based on content

## Example: Aftertouch to CC11 Converter

### Signal Flow
```
midiin → midiselect @touch → touchin → ctlout 11
       ↓
    midiselect @ctl 74 → (other processing) → midiout
       ↓
    (other messages) → midiout
```

### Key JSON Structure Elements
```json
{
  "patcher": {
    "boxes": [
      {
        "box": {
          "id": "obj-1",
          "maxclass": "newobj",
          "text": "touchin",
          "patching_rect": [x, y, width, height]
        }
      }
    ],
    "lines": [
      {
        "patchline": {
          "source": ["obj-1", outlet_index],
          "destination": ["obj-2", inlet_index]
        }
      }
    ]
  }
}
```

## Common Conversions

### Aftertouch → CC11
- Input: `touchin`
- Output: `ctlout 11`

### CC11 → Aftertouch
- Input: `ctlin 11`
- Output: `touchout`

### CC74 → CC11
- Input: `ctlin 74`
- Output: `ctlout 11`

## Important Notes

1. **DO NOT** create `.amxd` as a directory/bundle - it must be a binary file
2. **Always use `latin-1` encoding** when reading/writing the JSON portion
3. **Update the length header** after modifying JSON content
4. **Use symlinks** for development - changes to the source file reflect immediately in Ableton
5. **File size should be reasonable** (typically 4-10KB for simple MIDI processors)
6. If Ableton says "file exceeded maximum size", the binary format is corrupted

## Troubleshooting

### Device won't drag in Ableton
- Check file type with `file device.amxd` - should show "data"
- Verify file size is reasonable (< 100KB for simple devices)
- Check symlink points to correct file: `ls -la`

### "File exceeded maximum size" error
- Binary format is corrupted
- Re-create from a working template
- Ensure length header is correctly updated

### Device does nothing
- Open device in Max (right-click → Edit)
- Check object connections in patcher
- Verify correct MIDI objects are used

## Project Structure Example

```
max8-aftertouch-to-cc11/
├── Aftertouch_to_CC11.amxd          # Main device
├── CC11_to_Aftertouch.amxd          # Reverse converter
└── MAX_FOR_LIVE_CREATION_GUIDE.md   # This guide
```

With symlinks in:
```
~/Desktop/ABLETON LIVE SOUNDS/Library/User Library/Presets/MIDI Effects/Max MIDI Effect/
├── Aftertouch_to_CC11.amxd -> /path/to/project/Aftertouch_to_CC11.amxd
└── CC11_to_Aftertouch.amxd -> /path/to/project/CC11_to_Aftertouch.amxd
```

## Version Info

- Created: 2025-11-16
- Max Version: 7.3.3 (compatible with Max 8)
- Tested with: Ableton Live (various versions)
