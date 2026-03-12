# PDB Reader for GDB — Architecture & Data Flow

## Overview

PDB is a multi-stream file container where different streams provide different
debug information. Streams are composed of multiple blocks, which don't have to
be consecutive. The blocks are the actual physical parts of the file — the PDB
file itself consists of multiple fixed-size blocks (except for the header).

Following is the data from PDB we need to read on initialization.

### MSF header (SuperBlock):

The MSF SuperBlock is the first block in the PDB file and contains basic
information such as the block size, number of blocks, and most importantly,
the location of the stream directory, which is used to locate all other
streams in the file. The SuperBlock is 64 bytes long.

###  Stream directory

The stream directory is located immediately after the SuperBlock and specifies
which block belongs to which stream. Each stream can span multiple physical 
blocks that are not necessarily contiguous.

With the information from the stream directory, we are able to parse any stream.

###  PDB Info stream (stream 1)

Basic information stream - it's most significant part is the location of the 
the "/names" stream (the String Table) which contains the list of all the files
compiled into the PDB.

###  Names stream (String Table):

Contains info on all the files used by all modules compiled into the PDB.
The names are read out into the String Table. The table is loaded eagerly
because just about any module will need to reference it when trying to display
it's files. The concept of lazy loading assumes we access the data only when
needed i.e. - only when a particular module is referenced. In PDB case,
accessing just about any module (break, info sources...) will quickly reference 
this table in order to get the line information, thus we just preload the table.

###  DBI stream (stream 3):

DBI stream contains the debug information (line numbers, symbols, etc.) for all
the modules (object files) linked into the program. Each module's debug info is
in a different stream and we read those streams on request. Eagerly we only load
the header which contains info on per module streams (debug info is per module).

###  DBI File Info substream

Substream is just a piece of data located at a given offset in a stream.
The File Info substream contains info on all the files used by all the modules
compiled into the PDB - the Names Buffer. Names Buffer actually duplicates the
String Table but it also adds the information on files that go into each module.
This is suitable for Quick Functions that check if a files is in a module;
obtaining this info from the String Table would require expanding the parts
of the module stream, to get the sections that reference per per module files 
(indices into String Buffer).

The duplication of the file names likely exists for compatibility.

### TPI stream (stream 2)

The TPI (Type Program Information) stream contains all non-builtin type
records used by the program — pointers, modifiers, arrays, procedures, member
functions, structs, classes, unions, enums, bitfields, argument lists, etc.

A **type index** is a 32-bit integer that uniquely identifies a type. Indices
below 0x1000 are reserved for simple/builtin types (encoded within the index). 
Indices 0x1000 and above correspond to records in the TPI stream, assigned 
sequentially: the first record is 0x1000, the second 0x1001, etc. Symbol records 
and other type records reference types by their type index.

Each record in the stream has variable length consisting of a 2-byte `RecordLen`
, a 2-byte `RecordKind` (the "leaf type" identifier such as `LF_POINTER`, 
`LF_MODIFIER`, `LF_ARRAY`, `LF_PROCEDURE`, `LF_ARGLIST`...), and a payload whose 
layout depends on the leaf type. Fields within a record can reference other 
types by their type index, forming a directed graph (e.g. an `LF_POINTER` record 
contains the type index of the pointee type).

The TPI stream is parsed eagerly at load time — type records are indexed so
they can be resolved on demand when a symbol references a type index. Resolved
types are cached so each type index is converted to a GDB `struct type` at most
once.

### IPI stream (stream 4). TODO

The IPI (Id Program Information) stream has the same physical layout as the TPI
stream but contains *id records* rather than type records. Id records reference
items like functions, strings, and build information by name rather than by
type structure. Currently the IPI stream is not parsed.

### Module Streams
        
Module streams contain the debug information for individual modules (object
files). Various debug sections are specified using identifiers — e.g. symbols
or line information or file info. The line information is in C13 sections
(C11 sections are obsolete). C13 sections are split into subsections, most
importantly Checksums and Lines. The Checksums subsection references the
String Table to provide the source files that belong to the module, while the
Lines subsection maps addresses to source lines (analogous to `.debug_line` in
DWARF).

### Symbol Record Stream / GSI / PSGSI

The Symbol Record Stream (referenced by the DBI header) contains all global
symbol records — both private globals (S_GPROC32, S_GDATA32, S_PROCREF, etc.)
and public symbols (S_PUB32).

The PSGSI (Public Symbol Index) stream is PDB's equivalent of the ELF symbol
table (`.symtab`/`.dynsym`) — it contains a hash table whose hash records point
into the Symbol Record Stream to locate the S_PUB32 records stored there. 
After the hash table there is an address to name map that is used to build 
the GDB minimal symbol table.

The GSI (Global Symbol Index) stream is a hash table for O(1) name to symbol
lookup similar to DWARF's .debug_names. It indexes cross-reference records 
(S_PROCREF, S_LPROCREF, S_DATAREF) that point into module streams — each 
reference carries module index and offset, telling the reader which module 
contains the full symbol definition. We use this table to build the cooked index 
and provide quick functions for symbol lookup on GDB's request.

## Finding PDB files.

PDB files are searched at different locations - for the main executable the user
can specify the  --pdb-path command-line override. Further, we search for PDB
by the PDB name recorded in the so called RSDS record of the Debug Directory
section in the actual executable. This PDB name is searched as is or as the
base name in the EXE directory. We also search for the PDB by simply replacing
the EXE name.

Windows can specify the location of the PDB files in Windows registry or in the
environment variables.

TODO: For system DLLs, Windows normally uses so called Debug Symbol server from
where the PDB files can be downloaded.

## Path Conversion (MSYS2)

PDBs produced under MSYS2 can have Linux style paths which are converted into
Windows style paths before storing them to symtab linetables, so that GDB
can load them. This either requires prepending the MSYS2 root
(e.g. /home/PATH -> C:/msys2/PATH) or converting drive information
(e.g. /c/PATH -> C:/PATH).

The MSYS2 root must be specified using MSYS2_ROOT env. var, otherwise we look
into common msys2/mingw64 directories.


## Info Commands

All commands accept optional `path=<pdb-path>` and `modi=N` arguments to
select a specific PDB / module. If omitted, the default (main program) PDB and
all modules are used.

`info pdb-loaded-files`  List paths of all currently loaded PDB files.
`info pdb-modules`       List modules (object files) in the PDB with 
                         stream numbers and file counts.
`info pdb-files`         List source files per module from the DBI File Info 
                         substream.
`info pdb-files-c13`     List source files per module from C13 Checksums 
                         subsections, showing checksum type (MD5/SHA-1/SHA-256) 
                         and hash values.
`info pdb-lines`         Dump C13 line info: section:offset ranges and line 
                         number to offset mappings.
`info pdb-symbols`       Dump raw CodeView symbol records from module streams.
`info pdb-sym-records`   Dump records from the global symbol record stream.
`info pdb-gsi`           Dump GSI (Global Symbol Index) hash table: header, 
                         hash records, bitmap, bucket data. 
`info pdb-psi`           Dump PSGSI (Public Symbol Index) hash table with 
                         embedded GSI hash and address map.
`info pdb-locations`     Dump resolved variable location batons (ranges, 
                         register/offset, gaps). Requires `modi=N`; optional 
                        `symbol=NAME` to filter.

## Source Files

### `pdb.h`
Main header. Defines all public data structures, MSF/DBI/CodeView constants,
stream index numbers (PDB=1, TPI=2, DBI=3, IPI=4), and the public API.

**Key Structs:**

- `pdb_per_objfile` — Top-level context for one PDB file. Holds MSF geometry,
  stream directory, cached stream data, DBI data, module array, section
  addresses, string table, TPI context, GSI table, and the symbol record
  stream cache.
- `pdb_module_info` — Per-module metadata: stream number, symbol/C11/C13 byte
  sizes, section contribution, file lists (from File Info and from C13),
  expansion state and cached `compunit_symtab`.
- `pdb_tpi_context` — Parsed TPI stream: type record array, type cache.
- `pdb_tpi_type` — Single raw TPI record: leaf type, length, data pointer
  (into cached stream), data length.
- `pdb_gsi_hdr` — Parsed GSI hash header (signature, version, hash-record
  and bucket-data byte counts, data pointers, validity flag).
- `pdb_rsds_info` — RSDS record from the PE debug directory: GUID, age,
  PDB path.
- `pdb_loclist_baton` — Per-symbol location baton: linked list of location
  entries plus back-pointer to the PDB context.
- `pdb_loc_entry` — One DEFRANGE location range: start/end PC, register
  number, offset, flags, and inline gap array.
- `pdb_loc_gap` — Gap within a location entry (start/end addresses).
- `pdb_file_info` — Per-file checksum entry from C13: filename, checksum
  type and data.
- `pdb_line_block_info` — Callback data for `pdb_walk_c13_line_blocks`:
  filename, line section header, line array, line count.
- `CV_FileBlock`, `CV_FileChecksum` — On-disk C13 file block and checksum
  structures.
- `CV_LineSection`, `CV_Line` — On-disk C13 line section header and line
  entry.

**Key Functions:**

- `pdb_initialize_objfile()` — Entry point called from COFF reader; loads PDB,
                               expands modules, registers quick functions.
- `pdb_find_pdb_file()` — PDB file search.
- `pdb_read_stream()` — Read and cache an MSF stream by index.
- `pdb_read_tpi_stream()` — Parse TPI stream header and type records.
- `pdb_tpi_resolve_type()` — Resolve a type index to a GDB `struct type`.
- `pdb_build_module()` — Expand a single module into a `compunit_symtab`.
- `pdb_parse_symbols()` — Parse CodeView symbol records from a module stream.
- `pdb_read_sym_record_stream()` — Cache the global symbol record stream.
- `pdb_load_global_syms()` — Create GDB symbols from SymRecordStream globals.
- `pdb_parse_sym_record_stream()` — Parse/dump the SymRecordStream.
- `pdb_init_gsi_table()` — Parse GSI stream into a hash table.
- `pdb_build_minsyms()` — Create minimal symbols from PSGSI.
- `pdb_read_module_stream()` — Load a module's stream data.
- `pdb_read_module_files()` — Resolve file names from the File Info substream.
- `pdb_read_module_files_c13()` — Resolve file names from C13 checksums.
- `pdb_walk_c13_line_blocks()` — Walk C13 line blocks.
- `pdb_map_section_offset_to_pc()` — Convert (section, offset) to relocated PC.
- `pdb_register_loaded_pdb()` — Register a PDB for info commands.
- `pdb_init_loclist()` — Register PDB location list implementation with GDB.

### `pdb.c`
Core implementation. Handles MSF file I/O (reading blocks, assembling
streams), parsing the stream directory, /names stream (global string table),
DBI stream (module headers, section contributions), File Info substream,
and BFD section address mapping.  Walks C13 line subsections and builds GDB
symtabs with linetables.  Contains `pdb_initialize_objfile()` (the GDB entry
point), `pdb_expand_all_modules()`, and the `pdb_readnow_functions` quick
function table. Also handles MSYS2-style path conversion.

### `pdb-read-types.c`
TPI stream parser. Reads the TPI stream header and builds an indexed array of
`pdb_tpi_type` records — each record stores the leaf type, length, and a
pointer directly into the cached stream data (no copy). The type record array
is allocated on the objfile obstack.

A type cache (`struct type **`, 0x10000 entries covering all possible 16-bit
type indices) is also allocated on the objfile obstack. It maps type indices to
resolved GDB `struct type` pointers so each index is resolved at most once.
Simple/builtin types (0x0000–0x0FFF) and compound types (0x1000+) share the
same cache array. The cache is 512 KB on a 64-bit system.

Resolution is on-demand: when a symbol references a type index,
`pdb_tpi_resolve_type()` checks the cache, then either decodes the Kind+Mode
encoding (simple types) or parses the leaf record (LF_MODIFIER, LF_PROCEDURE,
LF_MFUNCTION, LF_POINTER, LF_ARRAY, LF_BITFIELD). Compound type resolution
is recursive — e.g. an LF_POINTER record references an underlying type index
that is itself resolved via the cache.

### `pdb-read-symbols.c`
CodeView symbol record parser. Contains `pdb_parse_symbols()` (per-module
symbol parsing), `pdb_load_global_syms()` (global symbol stream), and
`pdb_parse_sym_record_stream()` (DBI symbol record stream dump). Also contains
the `create_gdb_sym()` implementations for each CodeView symbol wrapper struct.

The function `pdb_loclist_read_variable` provides symbol (and it's location)
resolution to GDB by registering with GDB's symbol_computed_ops. Unlike DWARF
the implementation here uses LOC_COMPUTED for register variables as well, and
we use a single baton class.

All GDB symbols are allocated on the objfile obstack. Location batons
(`pdb_loclist_baton`) and their location entries (`pdb_loc_entry`, which
include an inline gap array) are also obstack-allocated. The symbol wrapper 
structs (`pdb_sym`) are stack-allocated during parsing — they exist only long 
enough to extract fields from the raw record and call `create_gdb_sym()`.

### `pdb-cv-regs-amd64.h`
CodeView register definitions for AMD64. Maps CodeView register IDs
(from Microsoft's `cvconst.h`) to DWARF and GDB register numbers.

### `pdb-path.c`
PDB file discovery. Searches for the PDB file using multiple strategies in
order: `--pdb-path` command-line override, PDB basename in the EXE directory
(from the RSDS record in the PE debug directory), EXE path with `.pdb`
extension, full RSDS path, `_NT_SYMBOL_PATH` / `_NT_ALT_SYMBOL_PATH`
environment variables, and Windows registry entries.

### `pdb-cmd.c`
GDB command registration. Implements all `info pdb-*` commands listed above.
Provides helper functions for parsing command arguments (`path=`, `modi=`)
and dispatching to the appropriate dump routines.

## GDB Integration

`pdb_initialize_objfile()` is the entry point, called from `coff_symfile_read()`
(in the COFF reader) before the DWARF initialization call. It calls 
`pdb_read_pdb_file()` to load and parse the PDB.

Each file registers the following with GDB:

- **`pdb.c`** — builds per-module symtabs with linetables and registers quick
  symbol functions (`pdb_readnow_functions`).
- **`pdb-read-symbols.c`** — Builds the CU; adds GDB symbols, function/scope 
  blocks, and symbol location info to the compute unit.
- **`pdb-read-types.c`** — Creates GDB types out of TPI types. 
- **`pdb-cmd.c`** — Registers `info pdb-*` commands for inspecting PDB internals.

## Initialization Order

`pdb_read_pdb_file()` loads PDB data in this order:

1. Validate MSF header (magic, block size, block count, directory location).
2. Read the stream directory (maps streams to blocks).
3. Read the /names stream (global string table for filenames).
4. Read the DBI stream (module headers, stream indices for GSI/PSGSI/SymRec).
5. Read the File Info substream (per-module file lists).
6. Read and parse the TPI stream (type records indexed for on-demand resolution).
7. Read PE section addresses from BFD (for section:offset → PC mapping).
8. Read and cache the Symbol Record Stream.
9. Build minimal symbols from PSGSI.
10. Register the PDB for info commands.
11. Expand all modules eagerly.

## Limitations / TODO

- No Windows x64 calling convention support.
- No struct/class/union/enum types. LF_STRUCTURE, LF_CLASS, LF_UNION,
  LF_ENUM are not yet resolved — returns void/unsupported placeholder.
  Variables of these types display as `<unsupported PDB type>`.
- No inline function support.
- Locals only accessible in current frame. `pdb_loclist_read_variable()`
  reads variables from live registers, so only the innermost frame (frame #0)
  is supported.  `up`/`down`/`frame N` need unwinding that is not yet supported.
- CodeView register mapping covers AMD64 GPRs only (RAX–R15, RSP, RBP).
  Other registers are not mapped — variables stored in those registers show as 
  unavailable.  Only x86-64 is supported.
- No IPI stream parsing.
- No language type detection.
- No MSVC name demangling. S_PUB32 records store mangled names, which appear in  
  `info pdb-psi` and minsyms. Module-level records store undecorated names so
  symbols display correct names.
- No PDB symbol server support (placeholder exists in `pdb-path.c`).
- No lazy loading — all modules expanded eagerly at load time.
- GSI table not yet used for lazy symbol lookup.
- Which API to expose to BFD ?
- Heap allocations - review for security - maybe limit the size.
- PDB file checksum checks.
- tests - switch from precompiled exe files to compilation.

## Memory Allocations

Most of the allocations use the objfile obstack.

Non obstack allocations:

**Heap (`new`):**
- `pdb_per_objfile` — registered via `registry<objfile>::key`, auto-deleted
  when objfile is destroyed.
- `buildsym_compunit` —  builder, deleted after modules are 
  `pdb_build_module()` / `pdb_expand_all_modules()`.

**Scoped (`unique_ptr<gdb_byte[]>`):**
- `pdb.c` - reading stream directory and stream block map.  Freed automatically.
- `pdb.c` `pdb_read_stream()` - reading of the actual streams bytes.
  Released into `pdb->stream_data[]` (`pdb` on objstack) or freed automaticaly.
- `pdb-path.c` — temporary buffers for PE executable access.
