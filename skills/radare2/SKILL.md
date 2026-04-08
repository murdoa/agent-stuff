---
name: radare2
description: "Reverse engineer binaries with radare2 CLI tooling. Inspect headers, functions, strings, symbols, cross-references, and decompile code from the terminal."
---

# radare2 Skill

Use `radare2` and companion tools like `rabin2` and `rax2` for terminal-first binary analysis. This is good for quick triage, embedded firmware poking, symbol/string extraction, xrefs, patching, and decompilation without needing a GUI.

Verified against `man radare2` on radare2 6.0.4.

## Prerequisites

Install radare2 using your platform package manager.

Examples:
```bash
nix shell nixpkgs#radare2
# or
nix profile install nixpkgs#radare2
# or
brew install radare2
```

Check availability:
```bash
r2 -v
rabin2 -v
```

## Quick Reference

| Task | Command |
|------|---------|
| Show binary info | `rabin2 -I ./binary` |
| List sections | `rabin2 -S ./binary` |
| List symbols/imports/exports | `rabin2 -s ./binary` / `rabin2 -i ./binary` / `rabin2 -E ./binary` |
| Extract strings | `rabin2 -zz ./binary` |
| Full auto-analysis | `r2 -A ./binary` |
| Reproducible startup (ignore user rc) | `r2 -N -A ./binary` |
| Deeper clean startup (ignore rc + plugins) | `r2 -NN -A ./binary` |
| Background analysis | `r2 -t ./binary` |
| Open with project | `r2 -p analysis.r2 ./binary` |
| Run scripts before/after load | `r2 -I pre.r2 -i post.r2 ./binary` |
| List functions | `r2 -Aqc 'afl' ./binary` |
| Print disassembly for current function | `r2 -Aqc 'pdf' ./binary` |
| Show decompilation (if plugin available) | `r2 -Aqc 's main; pdd' ./binary` |
| Find xrefs to symbol | `r2 -Aqc 'axt sym.imp.printf' ./binary` |
| Dump all function names JSON | `r2 -Aqc 'aflj' ./binary` |
| Seek to address | `r2 -q ./binary` then `s 0x401000` |

## Core Tools

### rabin2
Fast static metadata extraction without entering the interactive UI.

Common commands:
```bash
rabin2 -I ./firmware.bin        # binary info
rabin2 -S ./firmware.bin        # sections / segments
rabin2 -zz ./firmware.bin       # strings
rabin2 -s ./firmware.bin        # symbols
rabin2 -i ./firmware.bin        # imports
rabin2 -E ./firmware.bin        # exports
rabin2 -M ./firmware.bin        # main address / entry hints
```

### r2
Interactive analysis, disassembly, xrefs, patching, graphing, and scripting.

Start read-only when you are just inspecting:
```bash
r2 -A ./binary
```

For reproducible analysis that ignores your local radare config:
```bash
r2 -N -A ./binary
```

Useful flags:
- `-A` run `aaa` auto-analysis
- `-AA` run deeper `aaaa` analysis
- `-t` analyze binary info in a background thread
- `-q` quiet mode and quit after running commands
- `-c '<cmds>'` execute commands before the prompt
- `-i <file>` run a script after the file is loaded
- `-I <file>` run a script before the file is loaded
- `-p <project>` use a project file
- `-N` do not load user settings or rc scripts
- `-NN` do not load user settings, rc scripts, or plugins
- `-n` do not load binary info; treat as raw file
- `-nn` only load binary structures
- `-z` do not load strings
- `-zz` force loading strings even for raw files
- `-B <addr>` set base address for PIE binaries
- `-m <addr>` map file at given address
- `-e io.cache=true` enable cached writes before patching
- `-w` open in write mode

## Common Workflows

### 1. Fast Triage

```bash
rabin2 -I ./binary
rabin2 -S ./binary
rabin2 -i ./binary
rabin2 -zz ./binary
```

Use this first to identify architecture, entrypoint, linked libs, and obvious strings.

### 2. List and Inspect Functions

```bash
r2 -Aqc 'afl' ./binary
r2 -Aqc 's sym.main; pdf' ./binary
```

Useful commands inside `r2`:
```bash
afl         # list functions
s sym.main  # seek to main
pdf         # disassemble current function
pdr         # recursive-ish disassembly view
agf         # ascii graph for current function
axt         # xrefs to current address
axf         # xrefs from current address
```

### 3. Search for Interesting Strings and References

```bash
rabin2 -zz ./binary
r2 -Aqc '/ password; / secret; /j token' ./binary
```

Then inspect cross-references:
```bash
r2 -Aqc 's str.password; axt' ./binary
```

### 4. Find Imports and Their Callers

```bash
rabin2 -i ./binary
r2 -Aqc 'axt sym.imp.system' ./binary
r2 -Aqc 'axt sym.imp.strcpy' ./binary
```

Very good for spotting dangerous or juicy call sites in weird little goblin binaries.

### 5. JSON Output for Tooling

```bash
r2 -Aqc 'ij' ./binary              # file/bin info as JSON
r2 -Aqc 'aflj' ./binary            # functions JSON
r2 -Aqc 'izj' ./binary             # strings JSON
r2 -Aqc 'iij' ./binary             # imports JSON
r2 -Aqc 'iEj' ./binary             # exports JSON
r2 -Aqc 'agfj @ sym.main' ./binary # function graph JSON
```

Pipe to `jq` when needed.

### 6. Decompile

If the `r2dec` decompiler plugin is available:
```bash
r2 -Aqc 's main; pdd' ./binary
```

If `pdd` is unavailable, radare2 will error and suggest installing it via `r2pm -ci r2dec`.

If `pdd` is unavailable, fall back to disassembly and graph views:
```bash
r2 -Aqc 's main; pdf; agf' ./binary
```

### 7. Embedded / Raw Firmware Analysis

For raw blobs, specify architecture and base address explicitly.

Examples:
```bash
r2 -a arm -b 32 -m 0x08000000 -A ./firmware.bin
r2 -a mips -b 32 -e cfg.bigendian=true -A ./firmware.bin
```

Useful settings:
- `-a <arch>` architecture
- `-b <bits>` bit width
- `-m <addr>` map/load address
- `-B <addr>` base address for PIE binaries
- `-e cfg.bigendian=true` big-endian blobs

### 8. Reproducible Startup

For static analysis that should not depend on your personal `~/.radare2rc` weirdness:

```bash
r2 -N -A ./binary
```

If you also want to suppress automatic plugin loading:
```bash
r2 -NN -A ./binary
```

This is useful when comparing outputs across machines or trying to keep tooling behavior deterministic.

### 9. Projects

Use a project file when the analysis is nontrivial and you want to keep your flags, seeks, comments, and other session state around:

```bash
r2 -p analysis.r2 ./binary
```

For bigger binaries this saves you from re-deriving the same context like a goldfish with a disassembler.

### 10. Pre/Post-Load Scripts

Use scripts to make repeatable static-analysis workflows:

```bash
r2 -I pre.r2 -i post.r2 ./binary
```

Typical usage:
- `-I pre.r2` set eval vars before loading
- `-i post.r2` run analysis commands after loading

Good fit for standardizing analysis setup across samples.

### 11. Large Binary Analysis

For larger targets, background analysis can make startup feel less miserable:

```bash
r2 -t ./binary
```

You can combine it with reproducible startup too:
```bash
r2 -N -t ./binary
```

### 12. Patching

Only patch when the user explicitly wants modifications.

```bash
r2 -w ./binary
s 0x40123a
wa nop
wa jmp 0x401260
```

Safer variant with cache:
```bash
r2 -w -e io.cache=true ./binary
wc
q
```

## High-Signal Commands Inside r2

```bash
ii          # imports
iI          # binary info
iS          # sections
iz          # strings
afl         # functions
pdf         # disassemble function
pd 32       # disassemble 32 instructions
px 64       # hexdump 64 bytes
VV          # visual graph mode
s <addr>    # seek
/ <text>    # search text/bytes
/a mov eax  # search assembly pattern
axt         # xrefs to
axf         # xrefs from
afvn        # rename local vars
afn         # rename current function
```

## Suggested Analysis Flow

1. `rabin2 -I/-S/-i/-zz` for metadata triage
2. `r2 -Aqc 'afl'` to get the function map
3. inspect entrypoints / `sym.main` with `pdf`
4. follow imports and string xrefs with `axt`
5. use JSON commands for machine-readable extraction
6. only use write mode (`-w`) when patching is explicitly requested

## Troubleshooting

### No useful function names
Run deeper analysis:
```bash
r2 -AA ./binary
```

### Raw blob looks wrong
Set arch/bits/base address manually:
```bash
r2 -a arm -b 32 -m 0x0 -A ./blob.bin
```

### `pdd` does nothing
Decompiler plugin is probably missing. Use `pdf`, `pdr`, and `agf` instead.

### Analysis is noisy
Try quieter one-shots:
```bash
r2 -Aqc 'afl' ./binary
```

## Tips

- Prefer `rabin2` for fast non-interactive extraction.
- Prefer `r2 -Aqc '<cmd>'` when you need a single answer in scripts.
- Prefer `r2 -N` when you want reproducible results not polluted by local rc scripts.
- Use `-p` for any analysis session you might revisit; your future self is lazy and correct.
- Use `-i`/`-I` when you find yourself repeating the same setup commands.
- `-A` often emits relocation warnings on real ELF binaries; if xrefs/import behavior looks weird, try `-e bin.relocs.apply=true`.
- `-z` / `-zz` are useful when string loading is too noisy or when working with raw blobs.
- For firmware blobs, architecture assumptions are where you get owned first.
- `axt` on imports and interesting strings is absurdly high ROI.
- Stay read-only unless patching is explicitly requested.
