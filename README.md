# The CFF Table

OpenType supports embedding of CFF data blocks as defined in [Adobe's Tech Note 5176 ("The Compact Font Format Specification")](https://partners.adobe.com/public/developer/en/font/5176.CFF.pdf). For more information on this specification, please consult this Tech Note.

In addition to generic CFF, OpenType supports simplified CFF for font data that has been specifically intended to be used inside an OpenType wrapper. CFF data that is not intended to ever be used outside of OpenType context uses the four byte OpenType magic number instead of the four byte CFF Header. This simplification differs from the full CFF specification in the following way:

|CFF structure|Role in CFF|Role in OpenType-specific CFF|
|:---|:---|:---|
|Initial four bytes|CFF header|OpenType magic number|
|Name INDEX|Font names for all fonts in the CFF|*Not used*|
|Top DICT INDEX|DICT information that applies to all fonts in the CFF|Offsets to the rest of the CFF data|
|String INDEX|All strings needed for making the CFF human-readable|*Not used*|
|Global Subr INDEX|Global subroutines for all fonts|Font subroutines|
|Encodings|Encodings used by all fonts|*Not used*|
|Charsets|Character sets used by all fonts|*Not used*|
|FDSelect|glyph-to-FontDICT table used by CID fonts|*Not used*|
|CharStrings INDEX|Glyph outlines|Glyph outlines|
|Font DICT INDEX|Font DICT structures used by CID fonts|*Not used*|
|Private DICT|Font-specific DICT metadata|*Not used*|
|Local Subr INDEX|Font-specific subroutines|Secondary font subroutines|
|Copyrights and Trademarks|No longer used|*Not used*|



## Data Layout for an OpenType-specific CFF block

A CFF block intended purely for use inside an OpenType wrapped has the following data layout:

|Data structure|comments|
|:----|:-------------|
| OpenType magic number | *Must* be the four bytes `0x5F 0x0F 0x3C 0xF5`. |
| Top DICT INDEX | Encodes not-covered-by-OpenType metadata, and contains the offsets to the CharStrings and Local Subroutines structures. |
| Global Subr INDEX | *May* be used, *must* be a single byte 0x00 if not used. |
| CharStrings INDEX| Contains the glyph outlines for this font. *Must* use the Type 2 CharString format. |
| Local Subr INDEX | *May* be used, *must* be a single byte 0x00 if not used. |

## Data Types

Like all other OpenType data, OT-CFF data uses Motorola-style byte ordering (Big Endian). As a compact data format, OT-CFF does not incorporate any alignment restrictions, and as such there are no padding bytes used anywhere inside the CFF block itself, although the block as table may be padded to meet the alignment requirements of the OpenType wrapper.

Data objects are specified by byte offsets that are relative to some reference point within the CFF data. These offsets are 1 to 4 bytes in length. This documentation uses the convention of enclosing the reference point in parentheses and uses a reference point of (0) to indicate an offset relative to the start of the CFF data and (self) to indicate an offset relative to the first byte of the data structure containing the offset.

CFF uses five primary data formats, with Type 2 Charstrings specifying an additional data format for operator and operand information. In addition to the established OpenType Data Types (**OpenType spec section reference here?**), the following additional types are used: 

| Name | Range | Description|
|:---|:---|:---|
|Offset|variable|1, 2, 3, or 4 byte numeral representing a byte offset.|
|Offsize|1-4|1 byte unsigned numeral that specifies the size of (local) Offset(s)|
|DICT data|variable|semantic bytecode representing either DICT keys or numerals in several encodings|

### DICT data

DICT data encodes key-value pair represented in a compact tokenized format that is similar to that used to represent Type 2 charstrings. DICT keys are encoded as 1 or 2 byte operators, with dictionary values encoded as numeric operands that represent either integer or real values. An operator is preceded by the operand(s) that specify its value. A DICT is encoded as the concatenation of operand(s)/operator bytes.

#### Representing numbers in a DICT

Five different numeric encodings are defined, for efficiently representing various number ranges as well as real (as opposed to intenger) numbers. The following table shows these encodings, using the convention that the first byte of the operand is b0, the second byte is b1, and so on. The table is ordered by the number of bytes that need to be read to decode the number:

| Size | b0 range | value range | value calculation |
|:---|:---|:---|:---|
|1| 32-246 | -107 to 107 | `b0 - 139` |
|2| 247-250 | 108 to 1131 | `(b0-247) * 256 + b1 + 108` |
|2| 251-254 | -1131 to -108 | `-(b0-251) * 256 - b1 - 108` |
|3| 28 | –32768 to 32767 | `b1<<8 + b2` |
|5| 29 | -(2^31) to (2^31)-1 | `b1<<24 + b2<<16 + b3<<8 + b4` |

**Note**: *The 1, 2, and 3 byte integer formats used here are identical to those used in Type 2 CharStrings.*

Some examples of the integer formats are listed here.

|Value|Encoding|
|---:|:---|
|   0|`8b`|
| 100|`ef`|
|-100|`27`|
| 1000|`fa 7c`|
|-1000|`fe 7c`|
| 10000|`1c 27 10`|
|-10000|`1c d8 f0`|
| 100000|`1d 00 01 86 a0`|
|-100000|`1d ff fe 79 60`|

The real (as opposed to integer) number operand begins with a byte value of 30 (`0x1E`) followed by a variable-length sequence of bytes. Each byte encodes two 4-bit nibbles, with the first nibble of a pair stored in the most significant 4 bits of a byte, and the second nibble of a pair stored in the least significant 4 bits of a byte. The interpretation of each nibble is as follows: 

| Nibble | Represents |
|:---|:---|
|0-9| digit 0-9 |
|A| decimal point |
|B| positive exponent |
|C| negative exponent |
|D| *Not used* |
|E| minus sign |
|F| end of number |

A real number is terminated by one (or two) 0xF nibbles so that it is always padded to a full byte.

Some examples of the real formats are listed here.

|Value|Encoding|
|---:|:---|
|–2.25|`1E` for the operator, followed by `E2` (-2) `A2` (.2) `5F` (5, end of number)| 
|0.140541E–3|`1E` for the operator, followed by `0A` (0.) `140541` (140541) `C3` (E-3) `FF` (end of number)|

#### Representing operators in a DICT

Operators and operands may be distinguished by inspection of their first byte: 0–21 specify operators and 28, 29, 30, and 32–254 specify operands (numbers). Byte values 22–27, 31, and 255 are reserved values in DICT structures (unlike in Type 2 CharStrings).

OT-CFF follows the same stack restrictions as the generic CFF specification, such that an operator *may not* be preceded by more than 48 operands.

Operators may have one or more operands with an implicit format, formally declared in this specification for each operator. Possible operand interpretations are:

|Type|Description|
|:---|:---|
|number|Integer or real number|
|boolean|Integer type with the values 0 (false) and 1 (true)|
|array|A list of one or more numbers|
|delta|A number, or delta-encoded array of numbers (see below)|

The length of arrays is determined by counting the operands preceding the operator. 

The second and subsequent numbers in a delta-encoded array are encoded as the difference between successive values. For example, an array `a0, a1, ..., an` will be encoded as: `a0 (a1–a0) (a2–a1), ..., (an–a(n–1))`.

DICT operators may either use 1 or 2 bytes. 2-byte operators always start with the escape byte `12` (`0x0C`).

Unlike generic CFF, OT-CFF does not use default values for unspecified DICT keys. Most values for which generic CFF has default values are fields that are already encoded by OpenType font in OpenType tables, and as such should not be duplicated inside the CFF DICT. 

The list of DICT operators is as follows:

Operators that may be used in an OT-CFF Top DICT are:

|Operator byte code|Operator name|Description|Operand format|
|:---|:---|:---|:---|
|10|StdHW|...|number|
|11|StdVW|...|number|
|12 12|StemSnapH|...|number|
|12 13|StemSnapV|...|number|
|12 19|InitialRandomSeed|Seed value for the pseudo-random number generator (used if any of the CharStrings involve random numbers)|number (default=0)|
|17|CharStrings|Offset to the CharStrings INDEX|offset(0)|
|19|Subrs|Offset to the local subroutines|offset(self)|

The `defaultWidthX` and `nominalWidthX` operators supply width values for glyphs. If a glyph width equals the `defaultWidthX` value it can be omitted from the charstring, otherwise the glyph width is computed by adding the charstring width to `nominalWidthX` value. If `nominalWidthX` is carefully chosen the bulk of the widths in the charstrings can be reduced from 2-byte to single-byte numbers, thereby saving space.

**I'VE OMITTED THE `UniqueID` FIELD, BECAUSE FOR AN OPENTYPE-WRAPPED CFF THIS VALUE HAS NO MEANING** 

**FOR PURE OPENTYPE CFF, THE DEFAULT AND NOMINAL WIDTH VALUES ARE ALREADY COVERED BY PROPER OPENTYPE METRICS, AND SO HAVE BEEN OMITTED HERE**

**AS A HISTORICAL TYPE1 CONSTRUCT, I'VE TAKEN THE LIBERTY TO OMIT BLUE ZONE INFORMATION. IF, FOR SOME REASON, THIS SHOULD STILL BE PART OF A MODERN SPEC, THEN THEY WILL NEED EXPLICIT DESCRIPTIONS ON WHAT THEY'RE FOR AND WHY OPENTYPE DOESN'T COVER THIS, AS THEY ARE ABSOLUTELY NOT COVERED BY TECH NOTES 5176 AND 5177**

### INDEX data structures

An INDEX is an array of variable-sized objects. It comprises a header, an object data blob, and an offset array for locating entries inside the object data blob.

The offset array specifies offsets within the object data. An object is retrieved by indexing the offset array and fetching the object at the specified offset. The object’s length can be determined by subtracting its offset from the next offset in the offset array. An additional offset pointing to "the end of the array" is added at the end of the offset array so that the length of the last object may be determined by subtracting its offset value from this last offset. As such, An INDEX may be skipped by jumping to the offset specified by the last element of the offset array.

The INDEX structure is as follows:

|type|name|description|
|:---|:---|:---|
|USHORT|count|Number of objects stored in the INDEX's data region |
|Offsize|offSize|The number of bytes used for offsets in the offset array.|
|Offset[count+1]|offset|The array of offsets into the object data. Values start at 1 (see note below).|
|BYTE[]|data|Object data byte sequence|

Offsets in the offset array are relative to the byte that *precedes* the object data. As such, **the first element of the offset array always has value 1, not 0**. This ensures that every object has a corresponding offset which is always nonzero and permits the efficient implementation of dynamic object loading.

An empty INDEX is represented by a count field with a value "0". In this case, no additional fields need to be specified. Thus, the total size of an empty INDEX is 2 bytes with sequence `00 00`.

## OT-CFF uses the OpenType magic number instead of a CFF Header 

Where the generic CFF specification requires the CFF block to start with a four byte CFF header, the OT-CFF block must start with the four byte OpenType magic number `0x5F0F3CF5`.

Note that if a simplified CFF block is interpreted as generic CFF (for instance, if something copies the block out without rewriting it to proper generic CFF format for embedding in PostScript or PDF documents), the OpenType magic number will be interpreted as a CFF Header structure, leading to the following invalid parse result:

|Field|Value|
|:---|:---|
|Major version|95|
|Minor version|15|
|Header structure size in bytes|60|
|Global number of bytes used fo offset values|95|

This should lead to an explicit parse error because the `Offsize` field may only contain values 1, 2, 3, or 4. Thus, a simplified OpenType-CFF block will not work outside of an OpenType context.

## The Top DICT INDEX

The top-level dictionary INDEX is an INDEX structure following the byte layout explained in the secion on INDEX structures, and is used for describing several not-covered-by-OpenType values, as well as to locate the byte positions of the starts of the CharStrings INDEX (for glyph outlines) and local subroutines (which glyph outlines may rely on).

### Top DICT data

See the section on Data Types for which values may be encoded in the Top DICT's object data area.

## The Global Subr INDEX

This INDEX structure has an object data block that contains all (fragments of) Type 2 CharStrings that may be used as subroutines ("Subr") by glyphs defined in the CharStrings INDEX object data block.

If no global subroutines are used, this table must be encoded as a single byte 0 (`0x00`).

Sub routine numbers in Type 2 CharStrings are skewed by a number called the "subr number bias", which is calculated from the subroutine count in either the local or global subr INDEX structures. This bias is calculated as follows:

```
USHORT bias;
USHORT nSubrs = subrINDEX.count;

if (nSubrs < 1240)
  bias = 107;
else if (nSubrs < 33900)
  bias = 1131;
else
  bias = 32768;
```

For correct subroutine selection the calculated bias must be added to a subroutine's operand before accessing the appropriate subroutine INDEX.

This technique allows subroutine numbers to be specified using negative as well as positive numbers, thereby fully utilizing the available number ranges, thus saving space.

## The CharStrings INDEX

This INDEX structure has an object data block that contains all glyph outlines available to the OpenType font that this CFF block is housed in.

Glyph outlines *must* be defined using Type 2 CharStrings, and are accessed by GID as defined in the OpenType `cmap` table(s).

The first CharString (GID 0) *must* be the `.notdef` glyph. The number of glyphs encoded in the CharString INDEX object data block *must* be in agreement with the `numGlyphs` field in the `maxp` table.

### Type 2 CharStrings

The official specification for Type 2 Charstrings is [Adobe's Tech Note 5177, "The Type 2 CharString format"](https://partners.adobe.com/public/developer/en/font/5177.Type 2.pdf). Please consult this Tech Note for information on how to encode/decode data in Type 2 CharString format.

## The local Subr INDEX

This INDEX structure has an object data block that contains all (fragments of) Type 2 CharStrings that may be used as local subroutines by glyphs defined in the CharStrings INDEX object data block.

If no local subroutines are used, this table must be encoded as a single byte 0 (`0x00`).

Local subroutines, like global subroutines, require a "bias" correction before resolving. See the section on Global Subroutines for the description of how to calculate this bias. 
