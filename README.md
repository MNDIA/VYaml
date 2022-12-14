# VYaml


VYaml is a pure C# YAML 1.2 implementation, which is extra fast, low memory footprint with focued on .NET and Unity.

- The parser is heavily influenced by [yaml-rust](https://github.com/chyh1990/yaml-rust), and libyaml, yaml-cpp.
- Serialization interface/implementation heavily influenced by [Utf8Json](https://github.com/neuecc/Utf8Json), [MessagePack-CSharp](https://github.com/neuecc/MessagePack-CSharp), [MemoryPack](https://github.com/Cysharp/MemoryPack).

The reason VYaml is fast is it handles utf8 byte sequences directly with newface api set of C# (`System.Buffers.*`, etc).
In parsing, scalar values are pooled and no allocation occurs until `Scalar.ToString()`. This works with very low memory footprint and low performance overhead, in environments such as Unity.

![screenshot_benchmark_dotnet.png](./screenshots/screenshot_benchmark_dotnet.png)
![screenshot_benchmark_unity.png](./screenshots/screenshot_benchmark_unity.png)

Compared with [YamlDotNet](https://github.com/aaubry/YamlDotNet) (most popular yaml library in C#), basically 6x faster and about 1/50 heap allocations in some case.


## Currentry supported fetures

- Parser 
    - [YAML 1.2 mostly supported](#httpsyamlorgspec122)
- Deserialize
- Mainly focused on Unity
    - Only 2021.3 and higher (netstandard2.1 compatible)

## Most recent roadmap

- [ ] Support incremental source generator (Only Roslyn 4)
- Deserialize
    - [ ] Support `Stream`
    - [ ] Custom formatter
    - [ ] Restrict max depth
    - [ ] Interface-typed and abstract class-typed objects
    - [ ] Specific constructor
- [ ] Serialize

## Installation

### NuGet

Require netstandard2.1 or later.

You can install the following nuget package.
https://www.nuget.org/packages/VYaml

```bash
dotnet add package VYaml
```

### Unity

Require Unity 2021.3 or later. 

#### Install via git url

You can add following url to Unity Package Manager.

```
https://github.com/hadashiA/VYaml.git?path=VYaml.Unity/Assets/VYaml#0.1.0
```

## Usage

### Deserialize

Define a struct or class to be serialized and annotate it with the `[YamlObject]` attribute and the partial keyword.

```csharp
using VYaml.Annotations;

[YamlObject]
public partial class Sample
{
    // By default, public fields and properties are serializable.
    public string A; // public field
    public string B { get; set; } // public property
    public string C { get; private set; } // public property (private setter)
    public string D { get; init; } // public property (init-only setter)

    // use `[YamlIgnore]` to remove target of a public member
    [YamlIgnore]
    public int PublicProperty2 => PublicProperty + PublicField;
}

```

Why partial is necessary ?
- VYaml uses [SourceGenerator](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview) for metaprogramming, which supports automatic generation of partial declarations, sets to private fields.

The following yaml is deserializable to the above class `Sample`.

```yaml
a: hello
b: aaa
c: hoge
d: ddd
```

```csharp
var yamlUtf8Bytes = File.ReadAllBytes("path/to/yaml");
// var yamlUtf8Bytes = System.Text.Encofing.UTF8.GetBytes("<yaml string....>");
var sample = YamlSerializer.Deserialize<Sample>(yamlUtf8Bytes);
```

```csharp
sample.A // #=> "hello"
sample.B // #=> "aaa"
sample.C // #=> "hoge"
sample.D // #=> "ddd"
```

#### Deserialize as `dynamic`

You can also deserialize into primitive `object` type implicitly.

``` csharp
var yaml = YamlSerializer.Deserialize<dynamic>(yamlUtf8Bytes);
```

```csharp
yaml["a"] // #=> "hello"
yaml["b"] // #=> "aaa"
yaml["c"] // #=> "hoge"
yaml["d"] // #=> "ddd"
```

#### Naming convention

:exclamation: By default, VYaml maps C# property names in lower camel case (e.g. `propertyName`) format to yaml keys.

You can customize this behaviour with `[YamlMember("name")]`

```csharp
[YamlObject]
public partial class Sample
{
    [YamlMember("foo-bar")]
    public int FooBar { get; init; }
}
```

#### Built-in supported types

These types can be serialized by default:

- .NET primitives (`byte`, `int`, `bool`, `char`, `double`, etc.)
- Any enum (Currently, only simple string representation)
- `string`, `decimal`, `Half`
- `TimeSpan`, `DateTime`
- `Guid`, `Uri`
- `T[]`
- `Nullable<>`
- `List<>`
- `Dictionary<,>`
- `IEnumerable<>`, `ICollection<>`, `IList<>`, `IReadOnlyCollection<>`, `IReadOnlyList<>`
- `IDictionary<,>`, `IReadOnlyDictionary<,>`

TODO: We plan add more.


## Low-Level API

### Parser

`YamlParser` struct provides access to the complete meta-information of yaml.


- `YamlParser.Read()` reads through to the next syntax on yaml. (If end of stream then return false.)
- `YamlParser.ParseEventType` indicates the state of the currently read yaml parsing result.
- How to access scalar value:
    - `YamlParser.GetScalarAs*` families take the result of converting a scalar at the current position to a specified type.
    - Or we can use `YamlParser.TryGetScalarAs*` style.
- How to access meta information:
    - `YamlParser.TryGetTag(out Tag tag)` 
    - `YamlParser.TryGetCurrentAnchor(out Anchor anchor)`

Basic example:

```csharp
using var parser = YamlParser.FromBytes(utf8Bytes);

// YAML contains more than one `Document`. 
// Here we skip to before first document content.
parser.SkipAfter(ParseEventType.DocumentStart);

// Scanning...
while (parser.Read())
{
    // If the current syntax is Scalar, 
    if (parser.CurrentEventType == ParseEventType.Scalar)
    {
        var intValue = parser.GetScalarAsInt32();
        var stringValue = parser.GetScalarAsString();
        // ...
        
        if (parser.TryGetCurrentTag(out var tag))
        {
            // Check for the tag...
        }
        
        if (parser.TryGetCurrentAnchor(out var anchor))
        {
            // Check for the anchor...
        }        
    }
    
    // If the current syntax is Sequence (Like a list in yaml)
    else if (parser.CurrentEventType == ParseEventType.SequenceStart)
    {
        // We can check for the tag...
        // We can check for the anchor...
        
        parser.Read(); // Skip SequenceStart

        // Read to end of sequence
        while (!parser.End && parser.CurrentEventType != ParseEventType.SequenceEnd)
        {
             // A sequence element may be a scalar or other...
             if (parser.CurrentEventType = ParseEventType.Scalar)
             {
                 // ...
             }
             // ...
             // ...
             else
             {
                 // We can skip current element. (It could be a scalar, or alias, sequence, mapping...)
                 parser.SkipCurrentNode();
             }
        }
        parser.Read(); // Skip SequenceEnd.
    }
    
    // If the current syntax is Mapping (like a Dictionary in yaml)
    else if (parser.CurrentEventType == ParseEventType.MappingStart)
    {
        // We can check for the tag...
        // We can check for the anchor...
        
        parser.Read(); // Skip MappingStart

        // Read to end of mapping
        while (!parser.End && parser.CurrentEventType != ParseEventType.MappingEnd)
        {
             // After Mapping start, key and value appear alternately.
             
             var key = parser.GetScalarAsString();  // if key is scalar
             var value = parser.GetScalarAsString(); // if value is scalar
             
             // Or we can skip current key/value. (It could be a scalar, or alias, sequence, mapping...)
             // parser.SkipCurrentNode(); // skip key
             // parser.SkipCurrentNode(); // skip value
        }
        parser.Read(); // Skip MappingEnd.
    }
    
    // Alias
    else if (parser.CurrentEventType == ParseEventType.Alias)
    {
        // If Alias is used, the previous anchors must be holded somewhere.
        // In the High level Deserialize API, `YamlDeserializationContext` does exactly this. 
    }
}
```

See [test code](https://github.com/hadashiA/VYaml/blob/master/VYaml.Tests/Parser/SpecTest.cs) for more information.
The above test covers various patterns for the order of `ParsingEvent`.

## Customize serialization behaviour

- `IYamlFormatter<T>` interface customize the serialization behaviour of a your particular type.
- `IYamlFormatterResolver` interface can customize how it searches for `IYamlFormatter<T>` at runtime.

TODO:


## YAML 1.2 spec support status

### Implicit primitive type conversion of scalar

The following is the default implicit type interpretation.

Basically, it follows YAML Core Schema.
https://yaml.org/spec/1.2.2/#103-core-schema

|Support|Regular expression|Resolved to type|
|:-----|:-------|:-------|
| :white_check_mark: | `null \| Null \| NULL \| ~` | null |
| :white_check_mark: | `/* Empty */` | null |
| :white_check_mark: | `true \| True \| TRUE \| false \| False \| FALSE` | boolean |
| :white_check_mark: | `[-+]? [0-9]+` | int  (Base 10) |
| :x: | `0o [0-7]+` | int (Base 8) |
| :white_check_mark: | `0x [0-9a-fA-F]+` | int (Base 16) |
| :white_check_mark: | `[-+]? ( \. [0-9]+ \| [0-9]+ ( \. [0-9]* )? ) ( [eE] [-+]? [0-9]+ )?` | float |
| :white_check_mark: | `[-+]? ( \.inf \| \.Inf \| \.INF )` | float (Infinity) |
| :white_check_mark: | `\.nan \| \.NaN \| \.NAN` | float (Not a number) |

### https://yaml.org/spec/1.2.2/

Following is the results of the [test](https://github.com/hadashiA/VYaml/blob/master/VYaml.Tests/Parser/SpecTest.cs) for the examples from the  [yaml spec page](https://yaml.org/spec/1.2.2/).

- 2.1. Collections
  - :white_check_mark: Example 2.1 Sequence of Scalars (ball players)
  - :white_check_mark: Example 2.2 Mapping Scalars to Scalars (player statistics)
  - :white_check_mark: Example 2.3 Mapping Scalars to Sequences (ball clubs in each league)
  - :white_check_mark: Example 2.4 Sequence of Mappings (players’ statistics)
  - :white_check_mark: Example 2.5 Sequence of Sequences
  - :white_check_mark: Example 2.6 Mapping of Mappings
- 2.2. Structures
  - :white_check_mark: Example 2.7 Two Documents in a Stream (each with a leading comment)
  - :white_check_mark: Example 2.8 Play by Play Feed from a Game
  - :white_check_mark: Example 2.9 Single Document with Two Comments
  - :white_check_mark: Example 2.10 Node for “Sammy Sosa” appears twice in this document
  - :white_check_mark: Example 2.11 Mapping between Sequences
  - :white_check_mark: Example 2.12 Compact Nested Mapping
- 2.3. Scalars
  - :white_check_mark: Example 2.13 In literals, newlines are preserved
  - :white_check_mark: Example 2.14 In the folded scalars, newlines become spaces
  - :white_check_mark: Example 2.15 Folded newlines are preserved for “more indented” and blank lines
  - :white_check_mark: Example 2.16 Indentation determines scope
  - :white_check_mark: Example 2.17 Quoted Scalars
  - :white_check_mark: Example 2.18 Multi-line Flow Scalars
- 2.4. Tags
  - :warning: Example 2.19 Integers
  - :white_check_mark: Example 2.20 Floating Point
  - :white_check_mark: Example 2.21 Miscellaneous
  - :white_check_mark: Example 2.22 Timestamps
  - :white_check_mark: Example 2.23 Various Explicit Tags
  - :white_check_mark: Example 2.24 Global Tags
  - :white_check_mark: Example 2.25 Unordered Sets
  - :white_check_mark: Example 2.26 Ordered Mappings
- 2.5. Full Length Example
  - :white_check_mark: Example 2.27 Invoice
  - :white_check_mark: Example 2.28 Log File
- 5.2. Character Encodings
  - :white_check_mark: Example 5.1 Byte Order Mark
  - :white_check_mark: Example 5.2 Invalid Byte Order Mark
- 5.3. Indicator Characters
  - :white_check_mark: Example 5.3 Block Structure Indicators
  - :white_check_mark: Example 5.4 Flow Collection Indicators
  - :white_check_mark: Example 5.5 Comment Indicator
  - :white_check_mark: Example 5.6 Node Property Indicators
  - :white_check_mark: Example 5.7 Block Scalar Indicators
  - :white_check_mark: Example 5.8 Quoted Scalar Indicators
  - :white_check_mark: Example 5.9 Directive Indicator
  - :white_check_mark: Example 5.10 Invalid use of Reserved Indicators
- 5.4. Line Break Characters
  - :white_check_mark: Example 5.11 Line Break Characters
  - :white_check_mark: Example 5.12 Tabs and Spaces
  - :white_check_mark: Example 5.13 Escaped Characters
  - :white_check_mark: Example 5.14 Invalid Escaped Characters
- 6.1. Indentation Spaces
  - :white_check_mark: Example 6.1 Indentation Spaces
  - :white_check_mark: Example 6.2 Indentation Indicators
- 6.2. Separation Spaces
  - :white_check_mark: Example 6.3 Separation Spaces
- 6.3. Line Prefixes
  - :white_check_mark: Example 6.4 Line Prefixes
- 6.4. Empty Lines
  - :white_check_mark: Example 6.5 Empty Lines
- 6.5. Line Folding
  - :white_check_mark: Example 6.6 Line Folding
  - :white_check_mark: Example 6.7 Block Folding
  - :white_check_mark: Example 6.8 Flow Folding
- 6.6. Comments
  - :white_check_mark: Example 6.9 Separated Comment
  - :white_check_mark: Example 6.10 Comment Lines
  - :white_check_mark: Example 6.11 Multi-Line Comments
- 6.7. Separation Lines
  - :white_check_mark: Example 6.12 Separation Spaces
- 6.8. Directives
  - :white_check_mark: Example 6.13 Reserved Directives
  - :white_check_mark: Example 6.14 “YAML” directive
  - :white_check_mark: Example 6.15 Invalid Repeated YAML directive
  - :white_check_mark: Example 6.16 “TAG” directive
  - :white_check_mark: Example 6.17 Invalid Repeated TAG directive
  - :white_check_mark: Example 6.18 Primary Tag Handle
  - :white_check_mark: Example 6.19 Secondary Tag Handle
  - :white_check_mark: Example 6.20 Tag Handles
  - :white_check_mark: Example 6.21 Local Tag Prefix
  - :white_check_mark: Example 6.22 Global Tag Prefix
- 6.9. Node Properties
  - :white_check_mark: Example 6.23 Node Properties
  - :white_check_mark: Example 6.24 Verbatim Tags
  - :white_check_mark: Example 6.25 Invalid Verbatim Tags
  - :white_check_mark: Example 6.26 Tag Shorthands
  - :white_check_mark: Example 6.27 Invalid Tag Shorthands
  - :white_check_mark: Example 6.28 Non-Specific Tags
  - :white_check_mark: Example 6.29 Node Anchors
- 7.1. Alias Nodes
  - :white_check_mark: Example 7.1 Alias Nodes
- 7.2. Empty Nodes
  - :x: Example 7.2 Empty Content
  - :white_check_mark: Example 7.3 Completely Empty Flow Nodes
- 7.3. Flow Scalar Styles
  - :white_check_mark: Example 7.4 Double Quoted Implicit Keys
  - :white_check_mark: Example 7.5 Double Quoted Line Breaks
  - :white_check_mark: Example 7.6 Double Quoted Lines
  - :white_check_mark: Example 7.7 Single Quoted Characters
  - :white_check_mark: Example 7.8 Single Quoted Implicit Keys
  - :white_check_mark: Example 7.9 Single Quoted Lines
  - :white_check_mark: Example 7.10 Plain Characters
  - :white_check_mark: Example 7.11 Plain Implicit Keys
  - :white_check_mark: Example 7.12 Plain Lines
- 7.4. Flow Collection Styles
  - :white_check_mark: Example 7.13 Flow Sequence
  - :white_check_mark: Example 7.14 Flow Sequence Entries
  - :white_check_mark: Example 7.15 Flow Mappings
  - :white_check_mark: Example 7.16 Flow Mapping Entries
  - :white_check_mark: Example 7.17 Flow Mapping Separate Values
  - :white_check_mark: Example 7.18 Flow Mapping Adjacent Values
  - :white_check_mark: Example 7.20 Single Pair Explicit Entry
  - :x: Example 7.21 Single Pair Implicit Entries
  - :white_check_mark: Example 7.22 Invalid Implicit Keys
  - :white_check_mark: Example 7.23 Flow Content
  - :white_check_mark: Example 7.24 Flow Nodes
- 8.1. Block Scalar Styles
  - :white_check_mark: Example 8.1 Block Scalar Header
  - :x: Example 8.2 Block Indentation Indicator
  - :white_check_mark: Example 8.3 Invalid Block Scalar Indentation Indicators
  - :white_check_mark: Example 8.4 Chomping Final Line Break
  - :white_check_mark: Example 8.5 Chomping Trailing Lines
  - :white_check_mark: Example 8.6 Empty Scalar Chomping
  - :white_check_mark: Example 8.7 Literal Scalar
  - :white_check_mark: Example 8.8 Literal Content
  - :white_check_mark: Example 8.9 Folded Scalar
  - :white_check_mark: Example 8.10 Folded Lines
  - :white_check_mark: Example 8.11 More Indented Lines
  - :white_check_mark: Example 8.12 Empty Separation Lines
  - :white_check_mark: Example 8.13 Final Empty Lines
  - :white_check_mark: Example 8.14 Block Sequence
  - :white_check_mark: Example 8.15 Block Sequence Entry Types
  - :white_check_mark: Example 8.16 Block Mappings
  - :white_check_mark: Example 8.17 Explicit Block Mapping Entries
  - :white_check_mark: Example 8.18 Implicit Block Mapping Entries
  - :white_check_mark: Example 8.19 Compact Block Mappings
  - :white_check_mark: Example 8.20 Block Node Types
  - :white_check_mark: Example 8.21 Block Scalar Nodes
  - :white_check_mark: Example 8.22 Block Collection Nodes

## Credits

VYaml is inspired by:

- [yaml-rust](https://github.com/chyh1990/yaml-rust)
- [Utf8Json](https://github.com/neuecc/Utf8Json), [MessagePack-CSharp](https://github.com/neuecc/MessagePack-CSharp), [MemoryPack](https://github.com/Cysharp/MemoryPack)

## Aurhor

@hadashiA

## License

MIT

