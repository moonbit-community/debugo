# MoonBit Mach-O Library

A comprehensive MoonBit library for parsing Mach-O binary files, ported from Go's `debug/macho` package. This library provides full support for parsing Mach-O executables, dynamic libraries, object files, and bundles on macOS and iOS platforms.

## Features

- **Complete Mach-O Support**: Parse 32-bit and 64-bit Mach-O files
- **Cross-Architecture**: Support for x86, x86_64, ARM, ARM64, and PowerPC architectures
- **Load Command Parsing**: Handle all major load command types (segments, symbol tables, dynamic libraries, etc.)
- **Symbol Table Analysis**: Extract and analyze symbol tables and dynamic symbol tables
- **Section Analysis**: Parse and examine individual sections within segments
- **Utility Functions**: High-level convenience functions for common operations
- **Type Safety**: Leverages MoonBit's strong type system for safe binary parsing

## Quick Start

### Basic File Parsing

```moonbit
test "parsing a mach-o file" {
  // Create sample Mach-O header data (32-bit executable)
  let header_bytes = "\u{ce}\u{fa}\u{ed}\u{fe}" +  // magic_32 (little endian)
                     "\u{07}\u{00}\u{00}\u{00}" +  // cpu (i386 = 7)
                     "\u{00}\u{00}\u{00}\u{00}" +  // subcpu (0)
                     "\u{02}\u{00}\u{00}\u{00}" +  // file type (exec = 2)
                     "\u{00}\u{00}\u{00}\u{00}" +  // ncmd (0)
                     "\u{00}\u{00}\u{00}\u{00}" +  // cmdsz (0)
                     "\u{00}\u{00}\u{00}\u{00}"    // flags (0)

  let data = header_bytes.to_bytes()

  match @lib.parse_file(data) {
    Ok(file) => {
      println("Successfully parsed Mach-O file:")
      println("  Architecture: " + @lib.get_architecture(file))
      println("  File Type: " + @lib.get_file_type(file))
      println("  64-bit: " + @lib.is_64bit(file).to_string())
    }
    Err(err) => {
      println("Parse error: " + err.message)
    }
  }
}
```

### File Information Analysis

```moonbit
test "file analysis example" {
  let file = {
    header: {
      magic: @lib.magic_64,
      cpu: @lib.Arm64,
      sub_cpu: 0_U,
      type_: @lib.Exec,
      ncmd: 3_U,
      cmdsz: 200_U,
      flags: @lib.flag_pie.lor(@lib.flag_two_level)
    },
    byte_order: @lib.Little,
    loads: [],
    sections: [],
    symtab: None,
    dysymtab: None
  }

  // Get comprehensive file information
  let info = @lib.get_detailed_info(file)
  println("Detailed File Analysis:")
  println(info)

  // Check specific flags
  if @lib.has_flag(file, @lib.flag_pie) {
    println("This is a Position Independent Executable (PIE)")
  }

  // Get flag descriptions
  let flags = @lib.get_flags_description(file)
  println("File flags: " + @lib.join_strings(flags, ", "))
}
```

### Working with Constants and Enums

```moonbit
test "working with mach-o constants" {
  // File type detection
  let obj_type = @lib.Type::from_uint(1_U)
  inspect(obj_type, content="Object")

  let exec_type = @lib.Type::from_uint(2_U)
  inspect(exec_type, content="Exec")

  // CPU architecture detection
  let x86_cpu = @lib.Cpu::from_uint(7_U)
  inspect(x86_cpu, content="I386")

  let arm64_cpu = @lib.Cpu::from_uint(16777228_U)
  inspect(arm64_cpu, content="Arm64")

  // Load command types
  let segment_cmd = @lib.LoadCmd::from_uint(1_U)
  inspect(segment_cmd, content="Segment")

  let symtab_cmd = @lib.LoadCmd::from_uint(2_U)
  inspect(symtab_cmd, content="Symtab")
}
```

### Binary Utilities

```moonbit
test "binary parsing utilities" {
  // C-string extraction
  let data = "hello\u{00}world".to_bytes()
  let str = @lib.cstring(data)
  inspect(str, content="hello")

  // Reading integers with byte order
  let int_data = "\u{01}\u{02}\u{03}\u{04}".to_bytes()
  let little_endian = @lib.read_uint(int_data, 0, @lib.Little)
  let big_endian = @lib.read_uint(int_data, 0, @lib.Big)

  inspect(little_endian, content="67305985")   // 0x04030201
  inspect(big_endian, content="16909060")      // 0x01020304

  // Byte order detection from magic numbers
  let byte_order = @lib.determine_byte_order(@lib.magic_32)
  inspect(byte_order, content="Some(Little)")
}
```

## API Reference

### Core Types

#### `File`

Represents a parsed Mach-O file with all its components:

- `header: FileHeader` - File header information
- `byte_order: ByteOrder` - Endianness of the file
- `loads: Array[LoadCommand]` - All load commands
- `sections: Array[Section]` - All sections
- `symtab: Symtab?` - Symbol table (if present)
- `dysymtab: Dysymtab?` - Dynamic symbol table (if present)

#### `FileHeader`

Contains basic file metadata:

- `magic: UInt` - Magic number identifying file format
- `cpu: Cpu` - Target CPU architecture
- `type_: Type` - File type (executable, library, etc.)
- `ncmd: UInt` - Number of load commands
- `flags: UInt` - File flags

#### Enums

**`Type`** - Mach-O file types:

- `Object` - Object file (.o)
- `Exec` - Executable
- `Dylib` - Dynamic library (.dylib)
- `Bundle` - Bundle

**`Cpu`** - CPU architectures:

- `I386` - Intel 32-bit
- `Amd64` - Intel 64-bit
- `Arm` - ARM 32-bit
- `Arm64` - ARM 64-bit
- `Ppc`, `Ppc64` - PowerPC

**`LoadCmd`** - Load command types:

- `Segment`, `Segment64` - Memory segments
- `Symtab` - Symbol table
- `Dysymtab` - Dynamic symbol table
- `Dylib` - Dynamic library dependency
- `Rpath` - Runtime search path

### Parsing Functions

#### `parse_file(data: Bytes) -> ParseResult[File]`

Main entry point for parsing Mach-O files from raw bytes.

#### `new_file(data: Bytes) -> ParseResult[File]`

Alias for `parse_file` for convenience.

#### `open_file(path: String) -> ParseResult[File]`

Placeholder for file system access (not implemented).

### Utility Functions

#### File Analysis

- `is_64bit(file: File) -> Bool` - Check if file is 64-bit
- `is_32bit(file: File) -> Bool` - Check if file is 32-bit
- `get_architecture(file: File) -> String` - Get architecture name
- `get_file_type(file: File) -> String` - Get file type name
- `get_file_summary(file: File) -> String` - Get basic file info

#### Flag Analysis

- `has_flag(file: File, flag: UInt) -> Bool` - Check if flag is set
- `get_flags_description(file: File) -> Array[String]` - Get flag names

#### Content Analysis

- `get_imported_symbols(file: File) -> Array[String]` - Get imported symbols
- `get_imported_libraries(file: File) -> Array[String]` - Get library dependencies
- `find_segment(file: File, name: String) -> Segment?` - Find segment by name
- `find_section(file: File, name: String) -> Section?` - Find section by name

### Binary Utilities

#### String Processing

- `cstring(data: Bytes) -> String` - Extract null-terminated string
- `cstring_from_fixed_int_array(data: FixedArray[Int]) -> String` - Convert fixed array to string

#### Binary Reading

- `read_uint(data: Bytes, offset: Int, byte_order: ByteOrder) -> UInt` - Read 32-bit integer
- `read_uint64(data: Bytes, offset: Int, byte_order: ByteOrder) -> UInt64` - Read 64-bit integer
- `read_bytes(data: Bytes, offset: Int, length: Int) -> Bytes` - Extract byte slice

#### Validation

- `determine_byte_order(magic: UInt) -> ByteOrder?` - Detect endianness from magic
- `is_valid_magic(magic: UInt) -> Bool` - Validate magic number

## Constants

### Magic Numbers

- `magic_32: UInt = 0xfeedface_U` - 32-bit Mach-O magic
- `magic_64: UInt = 0xfeedfacf_U` - 64-bit Mach-O magic
- `magic_fat: UInt = 0xcafebabe_U` - Universal binary magic

### File Flags

Common file flags include:

- `flag_pie` - Position Independent Executable
- `flag_two_level` - Two-level namespace
- `flag_no_undefs` - No undefined symbols
- `flag_dyld_link` - Linked by dynamic linker

## Error Handling

The library uses a `ParseResult[T]` type for error handling:

```moonbit
enum ParseResult[T] {
  Ok(T)
  Err(FormatError)
}
```

`FormatError` provides detailed error information:

- `offset: Int64` - Byte offset where error occurred
- `message: String` - Human-readable error message
- `value: String?` - Optional problematic value

## Security Considerations

This library is designed for parsing trusted Mach-O files. When parsing untrusted input:

- Validate file size before parsing
- Set reasonable limits on load command counts
- Be aware that malformed files may cause parsing errors or consume significant resources
- Consider additional validation for security-sensitive applications

## Limitations

- File system access is not implemented (placeholder only)
- Some advanced Mach-O features may not be fully supported
- Primarily tested with common Mach-O file types

## Contributing

This library is a port of Go's `debug/macho` package. When contributing:

- Maintain compatibility with the original Go API where possible
- Add comprehensive tests for new functionality
- Follow MoonBit coding conventions
- Update documentation for any API changes

## License

Apache-2.0 - See LICENSE file for details.
