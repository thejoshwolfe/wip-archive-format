## Format Specification

### Data Types

#### varint

Variable-length integer; a 63-bit unsigned integer encoded in 1-9 bytes.
All bytes except the last one have a `1` as their first bit.
The unsigned integer is represented as the concatenation of the remaining 7 bit codewords in Big-Endian order (most-to-least significant).
The first byte must not be `0x80`.

The range of possible values is `0-0x7fffffffffffffff`.
All 63-bit integers have a canonical representation in this format.

Example pseudocode to read and write this integer type:

```c
uint64_t readVarint() {
  uint64_t value = 0;
  uint8_t b;
  do {
    if (value > 0xffffffffffffff) panic("varint too long");
    b = readByte();
    if (value == 0 && b == 0x80) panic("varint has leading zeros");
    value <<= 7;
    value |= b & 0x7f;
  } while ((b & 0x80) != 0);
  return value;
}
void writeVarint(uint64_t value) {
  if (value > 0x7fffffffffffffff) panic("too large for varint");
  uint8_t reverse_buffer[9];
  unsigned int length = 0;
  uint8_t continuation_mask = 0;
  do {
    reverse_buffer[length++] = (value & 0x7f) | continuation_mask;
    value >>= 7;
    continuation_mask = 0x80;
  } while (value > 0);
  do {
    writeByte(reverse_buffer[--length]);
  } while (length > 0);
}
```

#### u16be

Unsigned 16-bit integer in Big-Endian byte order (most-to-least significant).

### High-level structure

The high-level structure is the following sections in this order:

* `Header header`
* `Entry[n] entries`: Where `n` is some number `0` or more.
* `IndexHeader index_header`
* `IndexEntry[n] index_entries`: Where `n` is the same as the `n` from `entires`.
* `Footer footer`

The first section (`header`) begins at the start of the archive.
Each subsequent section starts immediately after the previous section.
The archive ends immediately after the last section (`footer`).

Note: Although `n` is not explicitly defined anywhere in this archive format,
it's possible to parse this format sequentially from start to finish unambiguously.

### Header

The `Header` is the 4-byte magic number: `0xe7, 0x30, 0x1e, 0xda`.

### Entry

Each `Entry` in `entries` has the following structure:

* `byte`: The value: `0x03`.
* `varint f`: The number of metadata fields in this `Entry`.
* `MetadataField[f] fields`: `f` repetitions of `MetadataField` belonging to this `Entry`.
* If `fields` contains a `entry_contents_size` field:
  * `byte[contents_size.size] contents`: The entry contents.
* Otherwise:
  * `ContentsChunk[?] contents_chunks`: This `Entry` effectively has a `contents` field equal to the concatenation of each `contents_segment` in `contents_chunks`.

### MetadataField

Each `MetadataField` has the following structure:

* `varint field_size`: The size in bytes of the remainder of this `MetadataField` after the end of `field_size`.
* `varint id`: An integer identifying the meaning of this `MetadataField`.
* `byte[?] data`: The remainder of this `MetadataField` has meaning depending on the value of `id`.

Here is a list of defined ids:

* `0`: `entry_contents_size`. This may only appear in in an `Entry`, not an `IndexEntry`. `data` has the following structure:
  * `varint size`: The size of `contents`.
* `1`: `index_entry_chunked_size`. This may only appear in in an `IndexEntry`, not an `Entry`, and only when the corresponding `Entry` does not contain an `entry_contents_size` `MetadataField`. `data` has the following structure:
  * `varint size`: The size of the corresponding `Entry`'s effective `contents`.
* `2`: `index_entry_contents_size`. This may only appear in in an `IndexEntry`, not an `Entry`, and only when the corresponding `Entry` contains an `entry_contents_size` `MetadataField`. `data` has the following structure:
  * `varint size`: The size of the corresponding `Entry`'s `contents`.
* `3`: `file_name`: `data` is a UTF-8 encoded string representing the `Entry`'s file name. The following restrictions apply:
  * The string must not contain any of the following characters: `<>:"\|?*`.
  * The string must not contain any codepoint less than `0x20`.
  * The `/` character may appear in the string, and its intended meaning is to be a path separator for describing a directory structure. After splitting the string into segments delimited by `/`, each segment has the following restrictions:
    * The segment must not be blank. (Note: this forbids leading and trailing slashes.)
    * The segment must not be `.` or `..`.
  * The size of `data` in bytes must be less than `0x10000`.
* `4`: `is_directory`: Indicates that this `Entry` represents a directory rather than a regular file.
  * The size of `data` must be `0`.
  * The size of this `Entry`'s `contents` must be `0`.
* `5`: `symlink`: Indicates that this `Entry` represents a symbolic link rather than a regular file. This may only appear when a `file_name` `MetadataField` is also present in this `fields` array (not in the corresponding `Entry` or `IndexEntry` `fields` array).
  * The size of this `Entry`'s `contents` must be `0`.
  * `data` is a UTF-8 encoded string representing the link target. It has all of the restrictions listed above for `file_name`, with the following modifications. After splitting the string into segments delimited by `/`:
    * `..` segments are allowed as long as they precede all other segments, and the number of `..` segments is less than the number of segments in the `file_name` after splitting it into segments delimited by `/`. (Note: this prevents the link target from "escaping the current working directory".)
    * A `.` segment is allowed as long as it is the only segment.

All the bytes in `data` must be accounted for in the above specified meanings.

### ContentsChunk

Each `ContentsChunk` has the following structure:

* `byte chunks_continue`: The value `0x00` for the last `ContentsChunk`, and `0x01` otherwise.
* If `chunks_continue` is `0x01`:
  * `byte[0x10000] contents_segment`
* Otherwise:
  * `u16be chunk_size`
  * `byte[chunk_size] contents_segment`

Note: Because `chunk_size` cannot encode the number `0x10000`,
there is exactly one possible way to divide a sequence of bytes of a given length into chunks.

### IndexHeader

The `IndexHeader` is the 1-byte value: `0x02`.

### IndexEntry

There is an `IndexEntry` for every `Entry`, and `index_entries[i]` corresponds to `entiries[i]` for each `i`.
Each `IndexEntry` has the following structure:

* `byte`: The value: `0x01`.
* `varint entry_offset`: `index_entries[i].entry_offset` is offset from the start of `entries` of `entires[i]` for each `i`.
* `varint f`: The number of metadata fields in this `IndexEntry`.
* `MetadataField[f] fields`: `f` repetitions of `MetadataField` belonging to this `IndexEntry`.

`index_entries[i].fields` must not contain any fields that are found in `entries[i].fields` for each `i`.

### Footer

The `Footer` has the following structure:

* `byte`: The value: `0x00`.
* `varint index_size`: The size in bytes of `IndexEntry[n]`.
