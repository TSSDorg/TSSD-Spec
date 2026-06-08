# TSSD open binary data SPEC

author: Brook Zhou(TSSDorg#hotmail.com)
Date:  2026-06-08T07:20:50.52Z UTC

## TSSD description 
TSSD is short name for **[Type][Size][Size][Data]**.

TSSD is an open binary data, designed for object/struct/class serial, switch, transfer and storage.

**TSSD data are Little Endian**

TSSD programing library should consider support both of Big and Little Endians. 

## TSSD data composed header and data
  Normally there are only one TSSD-header and one chunk data.
  chunk data means one byte data type(Ttype) with following data.
Ttype definition descripte as the table 1 below. 
  

###1. TSSD header
Format: [Magic-header][TSSD-Version][Tschema][sizet][Schema-string]

| item         | value             | length | desc                                                                          |
| ------------ | ----------------- | ------ | ----------------------------------------------------------------------------- |
| Magic header | ['T','S','S','D'] | 4bytes | 4 fixed charactar                                                             |
| TSSD version | 1                 | 2bytes | TSSD current version 1, may upgrade in future                                 |
| Tschema      |                   | 1byte  | schema begin                                                                  | 
| sizet        |                   | 2byte  | schema string length                                                          | 
| schema string|[string]           | sizet  | dynamic length string, user can define himself, used for data type validation |


###2. TSSD data format
  There are two data format: fixed-length data and dynamic length data.
  
2.1 fixed length data, format **[Ttype][data]**, descript in table 1.

2.2 dynamic length chunk data:**[Ttype][Sizet][Sizea][Data]** (our spec name TSSD is from it).
**Note: All sizet or sizea are int16(2bytes in little endian)**
both of them print on little endian like that: 123 => [123][0], but this doc print [sizet=123] for short.
  Sizet(2bytes): total size of current chunk, including the following Sizea and Data, but excluding itself.
  Sizea(2byts): additional size, depends the Ttype, descript in table 2.


### TSSD Ttype definition
**Note: Ttype is 1 int8(char) only, default is positive(>0),  negative(<0) means omit data.**
  such as:
  [Tint32] means a 32bits number, [-Tint32] means the type is Tint32, but this field unset, just as null in DB
####1. fixed length data(table 1)

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
| Tfloat32 | 20    | float          | 4             |                                          |
| Tfloat64 | 21    | double         | 8             |                                          ||

####2. dynamic length data(table 2)

| Ttype       | value | following data              | sizea desc                          |
| ----------- | ----- | --------------------------- | ----------------------------------- |
| Tstring     | 22    | [sizet][chars]              |                                     |
| Ttime       | 23    | [sizet][RFC3339Nano string] |                                     |
| Tschema     | 24    | [sizet][chars]              |                                     |
| Tarray      | 25    | [sizet][sizea][data]        |                                     |
| TmergeArray | 26    | [Ttype][sizet][sizea][data] | number of array elements            |
| Tobject     | 27    | [sizet][sizea][data]        | number of the object(struct) fields |
| Tdict       | 28    | [sizet][sizea][data]        | number of the dict(map) nodes       |
| Traw        | 29    | [sizet][data]               |                                     |
| Tuser       | 0x7F  | [sizet][user-define-data]   |                                     || 


###3. composed data
Tarray, TmergeArray, Tobject, Tdict are composed struct, that means they are composed from some of fixed or dynamic length data, 
they will expand one by one as the previou Type. the basic format is as that:
[Ttype][Sizet][SizeA][data]

####3.1 Tstring
format: [Tstring][sizet][data]
Tsting is simple, follow a string length, then the string data
```
string("Hello TSSD") => [Tstring][sizet=10]{"Hello TSSD"}
```

####3.2 Tschema
format: [Tschema][sizet][data]
Tschema works as Tstring.
**TSSD writer put schema in the header to specify the TSSD data schema.
TSSD reader should validate the schema if match with local, should block unmarsh if not.**
User can define the schema content themselves.
```
Tschema("MyStruct_V2-0f0eff09d0") => [Tstring][sizet=22]{"MyStruct_V2-0f0eff09d0"}
```

####3.3 Ttime
format: [Ttime][sizet][data]
Ttime works as Tsting.
it prensents for timestamp within RFC3339Nano format
```
[Ttime][sizet=39]{"2023-06-08 11:34:50.371381984 +0000 UTC"}
```

####3.4 Traw
format: [Traw][sizet][data]
Traw means raw data, TSSD just forward it without parse.

####3.5 Tuser
format: [Tuser][sizet][data]
Tuser means user define data, TSSD process as Traw now. 


####3.6 Tarray
format: [Tarray][sizet][sizea][data]
Tarray presents dynamic array, sizea means element number in the array.
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

####3.7 TmergeArray
format: [TmergeArray][Ttype][sizet][sizea][data]
TmergeArray is for the basic fixed length type array only, the following Ttype spec the real array element type.
sizea means element number.
such as: 
```
int16[]= {123, 456, 789}
=>
[Tarray][sizet=11][sizea=3][Tint16][123][Tint16][456][Tint16][789]
```
can also marshaled in TmergeArray:
```
=>
[TmergeArray][Tint16][sizet=11][sizea=3][123][456][789]
```
compare with Tarray, it omit the repeat element Ttype to save storage. 

####3.8 Tobject
format: [Tobject][sizet][sizea][data]
Tobject means a object/struct/class, sizea means files number in struct, data need expand one by one field, such as
```
struct { int32(123), string("foobar")}
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


####3.9 Tdict
format: [Tdict][sizet][sizea][data]
Tdict mashaled just as struct{key, value}[], sizea means map node count.
**Note: Tdict no guarantee the key not conflict or unique. **
such as
```
map{
    {int64(234): string("hello")},
    {int64(678): string("TSSD")},
} =>
[Tdict][sizet=35][sizea=2][Tint64][234][Tstring][sizet=5]{"hello"}[Tint64][678][Tsting][sizet=4]{"TSSD"}
```
| value         | length(bytes) | desc                       |
| ------------- | ------------- | -------------------------- |
| [Tdict]       | 1             | dict(map) begin            |
| [sizet=35]    | 2             | dict total size: 35        |
| [sizea=2]     | 2             | 2 map nodes                |
| [Tint64]      | 1             | first map node key type    |
| [234]         | 8             | int64 value                |
| [Tstring]     | 1             | first node value type      |
| [sizet=5]     | 2             | string length: 5           |
| {"hello"}     | 5             | string content: "hello"    |
| [Tint64]      | 1             | second map node key type   |
| [678]         | 8             | int64 value                |
| [Tstring]     | 1             | second node value type     |
| [sizet=5]     | 2             | string length: 5           |
| {"TSSD"}      | 4             | string content: "TSSD"     |
