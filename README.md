## Format Specification

The archive format takes one of two forms, either the Indexed Layout or the Streaming Layout.

### varint

Variable-length integer; a 63-bit unsigned integer encoded in 1-9 bytes.
All bytes except the last one have a `1` as their first bit.
The unsigned integer is represented as the concatenation of the remaining
7 bit codewords in Big-Endian order (most-to-least significant).
The first byte must not be `0x80`.

Note: The range of possible values is `0-0x7fffffffffffffff`,
and all 63-bit integers have a canonical representation in this format.

### u16be

Unsigned 16-bit integer in Big-Endian byte order (most-to-least significant).

### Indexed Layout

* `byte[4]`: The values: `0xe7, 0x30, 0x1e, 0xda`.
* `varint entry_count`
* `IndexEntry[entry_count] entries`
* `byte[?] contents_area`: The concatenation of each entry's contents.

### Streaming Layout

* `byte[4]`: The values: `0xe7, 0x30, 0x1e, 0xdb`.
* `StreamEntry[?] entries`

### IndexEntry

* `varint contents_size`: The size in bytes of this entry's contents in `contents_area`.
* `varint field_count`
* `MetadataField[field_count] fields`

### StreamEntry

* `varint field_count`
* `MetadataField[field_count] fields`
* `StreamContentsChunk[?] contents_chunks`

### StreamContentsChunk

* `byte chunks_continue`: The value `0x00` for the last `ContentsChunk`, and `0x01` otherwise.
* If `chunks_continue` is `0x01`:
  * `byte[0x10000] contents_segment`
* Otherwise:
  * `u16be chunk_size`
  * `byte[chunk_size] contents_segment`

Note: Because `chunk_size` cannot encode the number `0x10000`,
there is exactly one possible way to divide a sequence of bytes of a given length into chunks.

### MetadataField

* `varint id`: An integer identifying the meaning of this `MetadataField`.
* `varint data_size`: The size in bytes of `data`.
* `byte[?] data`: The remainder of this `MetadataField` has meaning depending on the value of `id`.

## Metadata

All metadata is optional.
All metadata fields must appear in numeric order.

### 0 - `file_name`

`data` is a UTF-8 encoded string representing the entry's file name. The following restrictions apply:

* The string must be a valid UTF-8 byte sequence.
* The string must not contain any of the following characters: `<>:"\|?*`.
* The string must not contain any codepoint less than `0x20`.
* The `/` character may appear in the string, and its intended meaning is to be a path separator for describing a directory structure. After splitting the string into segments delimited by `/`, each segment has the following restrictions:
  * The segment must not be blank. (Note: this forbids leading and trailing slashes.)
  * The segment must not be `.` or `..`.
* The size of `data` in bytes must be greater than `0` and less than `0x10000`.

### 1 - `is_directory`

Indicates that this entry represents a directory rather than a regular file.

* This entry must also have a `file_name` `MetadataField`.
* The size of `data` must be `0`.
* The size of this entry's contents must be `0`.

### 2 - `is_executable`

Indicates that this entry should have executable permissions.
This may be significant for text files that are intended to be run as shell scripts.

* This entry must also have a `file_name` `MetadataField`.
* This entry must *not* have an `is_directory` `MetadataField`.
* The size of `data` must be `0`.

### 3 - `symlink`

Indicates that this entry represents a symbolic link rather than a regular file.

* This entry must also have a `file_name` `MetadataField`.
* This entry must *not* have an `is_directory` `MetadataField`.
* This entry must *not* have an `is_executable` `MetadataField`.
* The size of this entry's contents must be `0`.
* `data` is a UTF-8 encoded string representing the link target. It has all of the restrictions listed above for `file_name`, with the following modifications. After splitting the string into segments delimited by `/`:
  * `..` segments are allowed as long as they precede all other segments, and the number of `..` segments is less than the number of segments in the `file_name` after splitting it into segments delimited by `/`. (Note: this prevents the link target from "escaping the current working directory".)
  * A `.` segment is allowed as long as it is the only segment.
