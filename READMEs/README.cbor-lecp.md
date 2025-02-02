# RFC8949 CBOR Stream Parser

|||
|---|---|---|
|cmake| `LWS_WITH_LECP`, `LWS_WITH_LECP_FLOAT`|
|Header| ./include/libwebsockets/lws-lecp.h|
|api-test| ./minimal-examples/api-tests/api-test-lecp/|
|test app| ./test-apps/test-lecp.c -> libwebsockets-test-lecp|

LECP is the RFC8949 CBOR stream parsing counterpart to LEJP for JSON.

The features are:

 - completely immune to input fragmentation, give it any size blocks of CBOR as
   they become available, 1 byte, or 100K at a time give identical parsing
   results
 - input chunks discarded as they are parsed, whole CBOR never needed in memory
 - nonrecursive, fixed stack usage of a few dozen bytes
 - no heap allocations at all, just requires ~500 byte context usually on
   caller stack
 - creates callbacks to a user-provided handler as members are parsed out
 - no payload size limit, supports huge / endless strings or blobs bigger than
   system memory
 - collates utf-8 text and blob payloads into a 250-byte chunk buffer for ease
   of access

## Type limits

CBOR allows negative integers of up to 64 bits, these do not fit into a `uint64_t`.
LECP has a union for numbers that includes the types `uint64_t` and `int64_t`,
but it does not separately handle negative integers.  Only -2^63.. 2^64 -1 can
be handled by the C types, the oversize negative numbers wrap and should be
avoided.

## Floating point support

Floats are handled using the IEEE memory format, it means they can be parsed
from the CBOR without needing any floating point support in the build.  If
floating point is available, you can also enable `LWS_WITH_LECP_FLOAT` and
a `float` and `double` types are available in the number item union.  Otherwise
these are handled as `ctx->item.u.u32` and `ctx->item.u.u64` union members.

Half-float (16-bit) is defined in CBOR and always handled as a `uint16_t`
number union member `ctx->item.u.hf`.

## Callback reasons

The user callback does not have to handle any callbacks, it only needs to
process the data for the ones it is interested in.

|Callback reason|CBOR structure|Associated data|
|---|---|---|
|`LECPCB_CONSTRUCTED`|Created the parse context||
|`LECPCB_DESTRUCTED`|Destroyed the parse context||
|`LECPCB_COMPLETE`|The parsing completed OK||
|`LECPCB_FAILED`|The parsing failed||
|`LECPCB_VAL_TRUE`|boolean true||
|`LECPCB_VAL_FALSE`|boolean false||
|`LECPCB_VAL_NULL`|explicit NULL||
|`LECPCB_VAL_NUM_INT`|signed integer|`ctx->item.u.i64`|
|`LECPCB_VAL_STR_START`|A UTF-8 string is starting||
|`LECPCB_VAL_STR_CHUNK`|The next string chunk|`ctx->npos` bytes in `ctx->buf`|
|`LECPCB_VAL_STR_END`|The last string chunk|`ctx->npos` bytes in `ctx->buf`|
|`LECPCB_ARRAY_START`|An array is starting||
|`LECPCB_ARRAY_END`|An array has ended||
|`LECPCB_OBJECT_START`|A CBOR map is starting||
|`LECPCB_OBJECT_END`|A CBOR map has ended||
|`LECPCB_TAG_START`|The following data has a tag index|`ctx->item.u.u64`|
|`LECPCB_TAG_END`|The end of the data referenced by the last tag||
|`LECPCB_VAL_NUM_UINT`|Unsigned integer|`ctx->item.u.u64`|
|`LECPCB_VAL_UNDEFINED`|CBOR undefined||
|`LECPCB_VAL_FLOAT16`|half-float available as host-endian `uint16_t`|`ctx->item.u.hf`|
|`LECPCB_VAL_FLOAT32`|`float` (`uint32_t` if no float support) available|`ctx->item.u.f`|
|`LECPCB_VAL_FLOAT64`|`double` (`uint64_t` if no float support) available|`ctx->item.u.d`|
|`LECPCB_VAL_SIMPLE`|CBOR simple|`ctx->item.u.u64`|
|`LECPCB_VAL_BLOB_START`|A binary blob is starting||
|`LECPCB_VAL_BLOB_CHUNK`|The next blob chunk|`ctx->npos` bytes in `ctx->buf`|
|`LECPCB_VAL_BLOB_END`|The last blob chunk|`ctx->npos` bytes in `ctx->buf`|

## CBOR indeterminite lengths

Indeterminite lengths are supported, but are concealed in the parser as far as
possible, the CBOR lengths or its indeterminacy are not exposed in the callback
interface at all, just chunks of data that may be the start, the middle, or the
end.

## Handling CBOR UTF-8 strings and blobs

When a string or blob is parsed, an advisory callback of `LECPCB_VAL_STR_START` or
`LECPCB_VAL_BLOB_START` occurs first.  The `_STR_` callbacks indicate the
content is a CBOR UTF-8 string, `_BLOB_` indicates it is binary data.

Payload is collated into `ctx->buf[]`, the valid length is in `ctx->npos`.

For short strings or blobs where the length is known, the whole payload is
delivered in a single `LECPCB_VAL_STR_END` or `LECPCB_VAL_BLOB_END` callback.

For payloads larger than the size of `ctx->buf[]`, `LECPCB_VAL_STR_CHUNK` or
`LECPCB_VAL_BLOB_CHUNK` callbacks occur delivering each sequential bufferload.
If the CBOR indicates the total length, the last chunk is delievered in a
`LECPCB_VAL_STR_END` or `LECPCB_VAL_BLOB_END`.

If the CBOR indicates the string end after the chunk, a zero-length `..._END`
callback is provided.

## Handling CBOR tags

CBOR tags are exposed as `LECPCB_TAG_START` and `LECPCB_TAG_END` pairs, at
the `_START` callback the tag index is available in `ctx->item.u.u64`.

## Parsing paths

LECP maintains a "parsing path" in `ctx->path` that represents the context of
the callback events.  As a convenience, at LECP context creation time, you can
pass in an array of path strings you want to match on, and have any match
checkable in the callback using `ctx->path_match`, it's 0 if no active match,
or the match index from your path array starting from 1 for the first entry.

|CBOR element|Representation in path|
|---|---|
|CBOR Array|`[]`|
|CBOR Map|`.`|
|CBOR Map entry key string|`keystring`|



## Comparison with LEJP (JSON parser)

LECP is based on the same principles as LEJP and shares most of the callbacks.
The major differences:

 - LEJP value callbacks all appear in `ctx->buf[]`, ie, floating-point is
   provided to the callback in ascii form like `"1.0"`.  CBOR provides a more
   strict typing system, and the different type values are provided either in
   `ctx->buf[]` for blobs or utf-8 text strtings, or the `item.u` union for
   converted types, with additional callback reasons specific to each type.

 - CBOR "maps" use `_OBJECT_START` and `_END` parsing callbacks around the
   key / value pairs.  LEJP has a special callback type `PAIR_NAME` for the
   key string / integer, but in LECP these are provided as generic callbacks
   dependent on type, ie, generic string callbacks or integer ones, and the
   value part is represented according to whatever comes.


# Writing CBOR

CBOR is written into a `lws_lec_pctx_t` object that has been initialized to
point to an output buffer of a specified size, using printf type formatting.

Output is paused if the buffer fills, and the write api may be called again
later with the same context object, to resume emitting to the same or different
buffer.

This allows bufferloads of encoded CBOR to be produced on demand.

|Format|Arg(s)|Meaning|
|---|---|---|
|`123`||unsigned literal number|
|`-123`||signed literal number|
|`%u`|`unsigned int`|number|
|`%lu`|`unsigned long int`|number|
|`%llu`|`unsigned long long int`|number|
|`%d`|`signed int`|number|
|`%ld`|`signed long int`|number|
|`%lld`|`signed long long int`|number|
|`123(...)`||literal tag and scope|
|`%t(...)`|`unsigned int`|tag and scope|
|`%lt(...)`|`unsigned long int`|tag and scope|
|`%llt(...)`|`unsigned long long int`|tag and scope|
|`[...]`||Array (fixed le if `]` in same format string)|
|`{...}`||Map (fixed len if `}` in same format string)|
|`<...>`||Container for indeterminite string frags|
|`'string'`||Literal text of known length|
|`%s`|`const char *`|NUL-terminated string|
|`%.*s`|`int`, `const char *`|length-specified string|
|`%.*b`|`int`, `const uint8_t *`|length-specified binary|
|`:`||separator between Map items (a:b)|
|`,`||separator between Map pairs or array items|
|`<space>`||separator between ambiguous sequences|

## Examples

```
	uint8_t buf[128];
	lws_lec_pctx_t cbw;
	int n = -1;

	lws_lec_init(&cbw, buf, sizeof(buf));
	lws_lec_printf(ctx, "%d", n);
```

The printf returns `LWS_LECPCTX_RET_FINISHED`, `ctx->used` is 1, and `buf[0]` is
set to `0x20`.

```
	...
	int args[3] = { 1, 2, 3 };

	lws_lec_printf(ctx, "{'a':%d,'b'[%d,%d]}", args[0], args[1], args[2]);
```
