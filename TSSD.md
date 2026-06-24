# TSSD open binary data SPEC

author: Brook Zhou(TSSDorg#hotmail.com)

Version: 1

Date:  2026-06-08T07:20:50.52Z UTC

## TSSD description 

TSSD is an open binary data, used for object/struct/class serialize, transfer, exchange or storage.

TSSD is short name for **[Type][Size][Size][Data]**.

TSSD designed simple, cross-platform and high performance

**TSSD data are Little Endian**

TSSD programing library should implement both of Big and Little Endians. 

## TSSD frame
  Format: [TSSD-header][TSSD-data]
  Normally there are only one TSSD-header and one chunk data within a TSSD frame.
  
  both header and data should following format: [Ttype][data]
  
  **Ttype is always an 1 byte length int8(char)**
  Ttype definition describe in the table 1 below.
  
### 1. TSSD header

Format: [Thead='T'][header="SSD"][Tversion='V'][version=1][Tschema='S'][sizet][schema-string]

| item         | value             | length | desc                                                                          |
| ------------ | ----------------- | ------ | ----------------------------------------------------------------------------- |
| Thead        | ['T']             | 1byte  | TSSD header begin                                                             | 
| header       | ['S','S','D']     | 3bytes | 3 fixed charactar                                                             |
| Tversion     | ['V']             | 1byte  | version begin                                                                 | 
| TSSD version | 1                 | 2bytes | TSSD current version: 1, may upgrade in future                                |
| Tschema      | ['S']             | 1byte  | schema begin                                                                  | 
| sizet        |                   | 2byte  | schema string length                                                          | 
| schema string|[string]           | sizet  | dynamic length string, user can define himself, used for data type validation |

**"TSSD" is the magic header**

### 2. TSSD data
There are 2 data formats, fixed-length data and dynamic length data.

2.1 fixed length data, format **[Ttype][data]**, describe in table 1.

2.2 dynamic length data:**[Ttype][SizeT][SizeA][Data]** (our spec name TSSD is from it).

  - SizeT(2bytes): Total size of current chunk, including the following SizeA and Data, but excluding itself.

  - SizeA(2bytes): Additional size, depends the Ttype, describe in table 2.

**Note: All SizeT or SizeA are int16(2bytes in little endian)**

sizet same with SizeT, sizea same with SizeA.

both of them print on little endian like that: 123 => [123][0], but this doc print [sizet=123] for short.


### 3. TSSD Ttype definition
**Note: Ttype is 1 int8(char) only, default is positive(>0),  negative(<0) means omit data, TSSD header always positive**

  E.g. : [Tint32] means a 32bits number, [-Tint32] means the type is Tint32, but this field unset, just as null in Database.

#### 3.1 fixed length data

 fixed length data are primary number, and the length are hinted by the Ttype, so TSSD omit it.
 Format: [Ttype][data], data is in Little Endian
Table 1:

| Ttype    | value | CPP type       | length(bytes) | range(hex)                               |
| -------- | ----- | -------------- | ------------- | ---------------------------------------- |
| Tbool    | 11    | bool           | 1             | true/false                               |
| Tint8    | 12    | char           | 1             | [-0x80,0x7F]                             |
| Tuint8   | 13    | unsigned char  | 1             | [0, 0xFF]                                |
| Tint16   | 14    | short          | 2             | [−0x8000,0x7FFF]                         |
| Tuint16  | 15    | unsigned short | 2             | [0,0xFFFF]                               |
| Tint32   | 16    | int32_t        | 4             | [-0x80000000,0x7FFFFFFF]                 |
| Tuint32  | 17    | uint32_t       | 4             | [0,0xFFFFFFFFFFFFFFFF]                   |
| Tint64   | 18    | int64_t        | 8             | [-0x8000000000000000,0x7FFFFFFFFFFFFFFF] |
| Tuint64  | 19    | uint64_t       | 8             | [0,0xFFFFFFFFFFFFFFFF]                   |
| Tfloat32 | 20    | float          | 4             | 32-bit IEEE 754                          |
| Tfloat64 | 21    | double         | 8             | 64-bit IEEE 754                          ||


#### 3.2 dynamic length data

 Format: [Ttype][sizet][sizea][data]
Table 2:

| Ttype       | value  | following data              | sizea desc                          |
| ----------- | -----  | --------------------------- | ----------------------------------- |
| Tstring     | 22     | [sizet][chars]              |                                     |
| Ttime       | 23     | [sizet][RFC3339Nano string] |                                     |
| Tenum       | 24     | [sizet][enum string]        |                                     |
| Tarray      | 25     | [sizet][sizea][data]        |                                     |
| Tarraym     | 26     | [Ttype][sizet][sizea][data] | count of array elements             |
| Tobject     | 27     | [sizet][sizea][data]        | count of the object(struct) fields  |
| Tdict       | 28     | [sizet][sizea][data]        | count of the dict(map) nodes        |
| Tdictk      | 29     | [Ttype][sizet][...]         |                                     |
| Tdictv      | 30     | [Ttype][sizet][...]         |                                     |
| Traw        | 31     | [sizet][data]               |                                     |
| Tschema     | 83('S')| [sizet][chars]              |                                     |
| Thead       | 84('T')| ["SSD"]                     | 4 bytes MagicHead: "TSSD"           |
| Tversion    | 86('V')| [version]                   |                                     |
| Tuser       | 0x7F   | [sizet][user-define-data]   |                                     ||


### 4. basic dynamic length data

basic dynamic length data format: [Ttype][sizet][data], including Tstring, Ttime, Tschema, Traw, Tuser:

#### 4.1 Tstring

format: [Tstring][sizet][data]
Tstring is simple, follow a string length, then the string data
```
string("Hello TSSD") => [Tstring][sizet=10]{"Hello TSSD"}
```

#### 4.2 Tschema

format: [Tschema][sizet][data]
Tschema works as Tstring, it is important for TSSD, as TSSD marshal data only without type name.
**TSSD peer(reader and writer) should parse with the same schema.
TSSD writer put schema in the header to specify the TSSD data schema.
TSSD reader validate the schema if match with local, should block unmarsh if not.**
User can define the schema content and validation algorithem themselves, even put a large json string with all object type names.
```
Tschema("MyStruct_V2-0f0eff09d0") => [Tschema][sizet=22]{"MyStruct_V2-0f0eff09d0"}
```

#### 4.3 Ttime

format: [Ttime][sizet][data]
Ttime works as Tstring.
it prensents for timestamp within RFC3339Nano format
```
[Ttime][sizet=39]{"2023-06-08 11:34:50.371381984 +0000 UTC"}
```

#### 4.4 Tenum

format: [Tenum][sizet][data]
Tenum works as Tstring.
it prensents for enum string
```
[Tenum][sizet=9]{"Color.Red"}
```

#### 4.5 Traw

format: [Traw][sizet][data]
Traw means raw data, TSSD just forward it without parse.

#### 4.6 Tuser

format: [Tuser][sizet][data]
Tuser means user define data, TSSD process as Traw now. 


### 5. composed data

Tarray, Tarraym, Tobject, Tdict are composed struct, that means they are composed from some of fixed or dynamic length data, 
**all of them can embed within others or themself. they will expand one by one when marshaling.**
the basic format is as that: [Ttype][SizeT][SizeA][data]
list their format detail below with a sample and its explain


#### 5.1 Tarray

format: [Tarray][sizet][sizea][data]
Tarray presents dynamic array, sizea means element count in the array.
```
string[] = {"f", "bar"} 
=>
[Tarray][sizet=12][sizea=2][Tstring][sizet=1]{"f"}[Tstring][sizet=3]{"bar"}
```

| value         | length(bytes) | desc                       |
| ------------- | ------------- | -------------------------- |
| [Tarray]      | 1             | array begin                |
| [sizet=12]    | 2             | array total size: 12       |
| [sizea=2]     | 2             | 2 elements                 |
| [Tstring]     | 1             | first element is string    |
| [sizea=1]     | 2             | the string contains 1 byte |
| {"f"}         | 1             | string[0] content: "f"     |
| [Tstring]     | 1             | second string begin        |
| [sizet=3]     | 2             | the string contains 3 byte |
| {"bar"}       | 3             | string[1] content: "bar"   |

#### 5.2 Tarraym

format: [Tarraym][Ttype][sizet][sizea][data]
Tarraym is for the basic fixed length type array only, the following Ttype spec the real array element type.
sizea means element count.
```
int16[]= {123, 456, 789}
=>
[Tarray][sizet=11][sizea=3][Tint16][123][Tint16][456][Tint16][789]
```
can also marshaled in Tarraym:
```
=>
[Tarraym][Tint16][sizet=8][sizea=3][123][456][789]
```
compare with Tarray, it omit the repeat element Ttype to save storage. 

#### 5.3 Tobject

format: [Tobject][sizet][sizea][data]
Tobject means a object/struct/class, sizea means fields count in struct, data need expand field one by one.
```
struct { int32(123), string("foobar") }
=>
[Tobject][sizet=15][sizea=2][Tint32][123][Tstring][sizet=5]{"foobar"}
```
| value         | length(bytes) | desc                       |
| ------------- | ------------- | -------------------------- |
| [Tobject]     | 1             | object begin               |
| [sizet=15]    | 2             | object total size: 15      |
| [sizea=2]     | 2             | 2 fields                   |
| [Tint32]      | 1             | first field is int32       |
| [123]         | 4             | int32 value                |
| [Tstring]     | 1             | second field string        |
| [sizet=5]     | 2             | the string contains 5 byte |
| {"foobar"}    | 5             | string content: "foobar"   |


#### 5.4 Tdict

format: [Tdict][sizet][sizea]{[Tdictk][k1][Tdictv][v1]}{[Tdictk][k2][Tictv][v2]}...
Tdict expand and mashaled as key value pair array
Tdictk marks a key begin and Tdictv marks a value begin
the real data of k1, v1, k2, v2 begin with another Ttype and follow above rules
sizea means map node count.
**Note: Tdict no guarantee the key order, not conflict or unique. **
```
map{
    {int64(234): string("hello")},
    {int64(678): string("world!")},
} =>
[Tdict][sizet=41][sizea=2][Tdictk][Tint64][234][Tdickv][Tstring][sizet=5]{"hello"}[Tdictk][Tint64][678][Tdictv][Tstring][sizet=6]{"world!"}
```
| value         | length(bytes) | desc                       |
| ------------- | ------------- | -------------------------- |
| [Tdict]       | 1             | dict(map) begin            |
| [sizet=41]    | 2             | dict total size: 41        |
| [sizea=2]     | 2             | 2 map nodes                |
| [Tdictk]      | 1             | a node key begin           |
| [Tint64]      | 1             | first map node key type    |
| [234]         | 8             | int64 value                |
| [Tdictv]      | 1             | a node value begin         |
| [Tstring]     | 1             | first node value type      |
| [sizet=5]     | 2             | string length: 5           |
| {"hello"}     | 5             | string content: "hello"    |
| [Tdictk]      | 1             | a node key begin           |
| [Tint64]      | 1             | second map node key type   |
| [678]         | 8             | int64 value                |
| [Tdictv]      | 1             | a node value begin         |
| [Tstring]     | 1             | second node value type     |
| [sizet=6]     | 2             | string length: 6           |
| {"world!"}    | 6             | string content: "world!"   |

#### 5.4 Tdictk and Tdictv

Tdictk and Tdictv are used for mark map's key and value begin,
the real data format determinted by the following Ttype after them
see example at 5.3 Tdict

## Tips

1. be careful with the type that language or platform dependable, if you need TSSD data share cross platform.
- int: length may vary in 32bits and 64bit.
- unsigned number: Java does't implement it.
- Tdict(map) key attribute may vary with the language: Go accepts basic type only, while CPP can accept complex struct.
