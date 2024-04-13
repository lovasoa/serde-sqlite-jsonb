# serde-sqlite-jsonb

This crate provides a custom Serde deserializer for SQLite JSONB columns.

It was initially developed for inclusion in the
[SQLPage](https://github.com/lovasoa/SQLpage) website builder.

## Why

Since version 3.45.0, SQLite supports JSONB columns,
which can store JSON data in a binary format that is more efficient
to manipulate than JSON.

The problem is that applications that use SQLite currently need
to convert the data from JSONB to JSON, and then from JSON to their
own data structures to use it.
This prevents reading the blob data directly from the
database in a streaming fashion using SQLite's blob streaming API,
and requires making a SQL query to extract and convert the data to JSON.

This crate provides a custom Serde serializer and deserializer
for JSONB directly, which allows skipping the JSON conversion step.

This can lead to significant performance improvements in certain scenarios,
as demonstrated in this crate's benchmarks.

### Benchmarks

These graphs show the time taken:
 - deserializing a JSONB column directly to a struct using this crate
 - making a SQL query to extract the JSONB column as JSON and then deserializing it to a struct using serde_json.

The data being deserialized contains a string, the length of which varies from 50 to 1000 characters, to demonstrate the evolution of the performance with the size of the data.

<center>

![Benchmark results](./benches/lines.svg)
![Benchmark results](./benches/violin.svg)

</center>

> *Disclaimer*: These benchmarks should always be taken with a grain of salt.
> When performance matters, you should measure the performance
> inside your own application with your own data.
> `serde_json` is very well optimized and can be faster than this crate
> in some scenarios, especially when the JSON data is small.

## Crate features

The binary format can contain raw json data, so this crate depends on the `serde_json` crate to parse the JSON data.
Since SQLite also supports json5, the `serde-json5` feature can be used if json5 support is needed.

By default, the (faster) `serde_json` feature is enabled and this crate returns an error when trying to parse json5 data.
To enable json5 support, enable the `serde-json5` feature
(and optionally disable the default features to use the json5 parser even for json data):

```toml
[dependencies]
serde-sqlite-jsonb = { version = "0.1", features = ["serde-json5"], default-features = false }
```

## Usage

This library does not handle the SQLite connection,
so you need to use a crate like `rusqlite` or `sqlx` to interact with the database.

Once you have extracted the JSONB data from the database,
either as a `Vec<u8>` or as a `std::io::Read` object that streams the BLOB data
from the database, you can use the `serde_sqlite_jsonb` crate to deserialize the JSON data,
either to your own data structures or to a `serde_json::Value`.

### Deserialize JSONB from a query result

```rust
let conn = rusqlite::Connection::open_in_memory()?;
let blob: Vec<u8> = conn.query_row(
    r#"select jsonb('{"id": 1, "name": "John Doe"}')"#, [], |row| row.get(0),
)?;
let person: Person = serde_sqlite_jsonb::from_bytes(&blob).unwrap();
```

### Streaming deserialization from a SQLite BLOB

```rust
let my_blob = conn.blob_open( // returns an object that implements std::io::Read
    DatabaseName::Main,
    "my_table", // table name
    "my_jsonb_column", // column name
    42, // primary key (rowid)
    true // read-only
)?;
let parsed: serde_json::Value = // or any other type that implements Deserialize
    serde_sqlite_jsonb::from_reader(my_blob).unwrap();
```

## Format

The format of the JSONB column is described in the SQLite documentation:
https://sqlite.org/draft/jsonb.html

The data format is a binary format with a header and payload. The header contains information about the type of the element and the size of the payload. The payload contains the actual data.

Here's a rough ASCII representation:

```
bits:  0  1  2  3  4  5  6  7  8
    +-------------+-------------+
    |  size(4)    | type(4)     | first header byte
    +-------------+-------------+
    |   payload size (0 - 64)   | header bytes number 2 to 9
    +---------------------------+
    |   payload data            | payload bytes (JSON strings or numbers in text format)
    +---------------------------+
```

### Header
#### Payload size

If the payload data is between 0 and 11 bytes (inclusive), the size is encoded in the first 4 bits of the header.
Otherwise, the size of the payload is encoded in the next bytes, and the first 4 bits indicate the number of bytes used to encode the payload size, using the following table:

| Payload Data Size Range | Size Encoding | First 4 bits of Header |
|-------------------------|---------------|------------------------|
| 0 to 11 bytes           | u4 (embedded in first 4 bits)  | 0 to 11 (0x0 to 0xB)             |
| 12 to 2^8 - 1 bytes     | u8     | 12 (0xC)               |
| 2^8 to 2^16 - 1 bytes   | u16 | 13 (0xD)               |
| 2^16 to 2^32 - 1 bytes  | u32 | 14 (0xE)              |
| 2^32 to 2^64 - 1 bytes  | u64 | 15 (0xF)             |

#### Type

| Type         | Hex Code | Description |
|--------------|----------|-------------|
| `Null`       | 0x0      | The element is a JSON "null". |
| `True`       | 0x1      | The element is a JSON "true". |
| `False`      | 0x2      | The element is a JSON "false". |
| `Int`        | 0x3      | The element is a JSON integer value in the canonical RFC 8259 format. |
| `Int5`       | 0x4      | The element is a JSON5 integer, such as `0xABC`. |
| `Float`      | 0x5      | The element is a JSON floating-point value in the canonical RFC 8259 format. |
| `Float5`     | 0x6      | The element is a JSON5 floating-point value that is not in the canonical JSON format. |
| `Text`       | 0x7      | The element is a JSON string value that does not contain any escapes. |
| `TextJ`      | 0x8      | The element is a JSON string value that contains RFC 8259 character escapes. |
| `Text5`      | 0x9      | The element is a JSON5 string value that contains character escapes, including some from JSON5. |
| `TextRaw`    | 0xA      | The element is a JSON string value that contains UTF8 characters that need escaping in JSON. |
| `Array`      | 0xB      | The element is a JSON array. The header of the first array element starts immediately after the array header. |
| `Object`     | 0xC      | The element is a JSON object. Object keys (strings) and values are alternated in the payload. |
| `Reserved13` | 0xD      | Reserved for future expansion. |
| `Reserved14` | 0xE      | Reserved for future expansion. |
| `Reserved15` | 0xF      | Reserved for future expansion. |

#### Example

The following JSON object:
```json
{"a": false, "b":true}
```
is encoded as the following 7 bytes of binary data:

```
6c 17 61 02 17 62 01
```

byte | value | description
-----|-------|-------------
0    | 0x6c  | header: payload size = 6, type = Object (0xC)
1    | 0x17  | header: payload size = 1, type = Text (0x7)
2    | 0x61  | payload: 'a'
3    | 0x02  | header: payload size = 0, type = False (0x2)
4    | 0x17  | header: payload size = 1, type = Text (0x7)
5    | 0x62  | payload: 'b'
6    | 0x01  | header: payload size = 0, type = True (0x1)


## MSRV

Requires rust >= 1.63 (debian stable)

## Profile-Guided Optimization (PGO)

According to the [tests](https://github.com/lovasoa/serde-sqlite-jsonb/discussions/1#discussion-6500343), PGO helps with achieving better performance - up to ~20% speed up. So if you want to optimize the library usage - consider enabling PGO for your application.
