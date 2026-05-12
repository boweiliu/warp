# Warp Code Editor Vim Keybindings: Reference & Test Coverage

## Vim Keybindings Reference

Source: [Code Editor Vim Keybindings - Warp docs](https://docs.warp.dev/code/code-editor/code-editor-vim-keybindings)

The code editor starts in Normal mode. Only "Exit Vim Insert Mode" is customizable.

### Movement

| Key | Action |
|-----|--------|
| `h`, `j`, `k`, `l` | Single-character movement |
| `<space>`, `<backspace>` | Single-character movement with line wrap |
| `w`, `W`, `b`, `B`, `e`, `E` | Word / WORD movement |
| `ge`, `gE` | End of previous word / WORD |
| `$` | End of line |
| `0` | Beginning of line |
| `^` | First non-whitespace character |
| `_` | Beginning of current line |
| `+` | First non-whitespace of next line |
| `-` | First non-whitespace of previous line |
| `%` | Jump to matching bracket |
| `[`, `]` | Previous/next unmatched bracket |
| `{`, `}` | Previous/next paragraph |
| `gg`, `G` | Jump to first/last line |

### Editing

| Key | Action |
|-----|--------|
| `r` | Replace character under cursor |
| `d`, `D` | Delete range/object or to end of line |
| `c`, `C` | Change range/object (delete then insert) |
| `s`, `S` | Substitute character/line |
| `x`, `X` | Delete under/before cursor |
| `y`, `Y` | Yank (copy) |
| `p`, `P` | Paste after/before cursor |
| `u`, `⌃r` | Undo / redo |
| `~` | Toggle case under cursor |
| `gu` | Lowercase |
| `gU` | Uppercase |
| `J` | Join lines |
| `.` | Repeat last edit |
| `gcc` | Toggle comments on line |
| `gc` | Toggle comments on selection |

### Text Objects

Used with operators (`d`, `c`, `y`, etc.) or visual mode:

| Prefix | Meaning |
|--------|---------|
| `i` | Inner (exclude delimiters) |
| `a` | Around (include delimiters) |

| Object | What it selects |
|--------|----------------|
| `w`, `W` | Word / WORD |
| `"`, `'`, `` ` `` | Quote-delimited text |
| `(`, `)` | Parenthesized text |
| `{`, `}` | Brace-delimited text |
| `[`, `]` | Bracket-delimited text |

### Search

| Key | Action |
|-----|--------|
| `f`, `F` | Find next/previous character on line |
| `t`, `T` | Find next/previous character (cursor before) |
| `;` | Repeat character search same direction |
| `,` | Repeat character search opposite direction |
| `/`, `?`, `*`, `#` | Opens Warp command search (not buffer search) |

### Mode Switching

| Key | Action |
|-----|--------|
| `i` | Insert before cursor |
| `I` | Insert at line start |
| `a` | Append after cursor |
| `A` | Append at line end |
| `o` | New line below, enter insert |
| `O` | New line above, enter insert |
| `v` | Visual character mode |
| `V` | Visual line mode |

### Registers

Register prefix `"` accesses:

| Register | Description |
|----------|-------------|
| `a`–`z`, `A`–`Z` | Named registers |
| `+`, `*` | System clipboard |
| `"` | Unnamed register (last delete/yank) |

---

## Unit Test Coverage

**9 test files, 204 total tests** across the codebase.

### Core Vim FSA (`crates/vim/`)

| File | Tests | Coverage |
|------|-------|---------|
| `src/matching_brackets_tests.rs` | 1 | `%` bracket matching — parentheses, square brackets, curly braces, nested and multi-line |
| `src/word_iterator_tests.rs` | 9 | Word motions: `w`, `W`, `b`, `B`, `e`, `E`, `ge`, `gE`, out-of-bounds edge case |
| `src/paragraph_iterator_tests.rs` | 6 | Paragraph motions: `{`, `}`, file boundaries, consecutive blank lines |

### Text Objects (`crates/vim/src/text_objects/`)

| File | Tests | Coverage |
|------|-------|---------|
| `word_tests.rs` | 4 | `iw`, `aw`, `iW`, `aW` |
| `block_tests.rs` | 2 | `i(`, `a(`, `i{`, `a{`, `i[`, `a[` |
| `quote_tests.rs` | 2 | `i'`, `a'`, `i"`, `a"`, `` i` ``, `` a` `` |
| `paragraph_tests.rs` | 12 | `ip`, `ap` — empty buffer, single/multi paragraph, blank lines, trailing blanks, whitespace-only lines |

### Code Editor Integration (`app/src/code/editor/view/`)

| File | Tests | Coverage |
|------|-------|---------|
| `vim_handler_tests.rs` | 44 | Mode switching, numbered repeats, character/word/line motions, delete/change/yank operators, text objects, visual selections, register operations, paste, jump commands (`gg`, `G`) |

### Terminal Editor Integration (`app/src/editor/view/`)

| File | Tests | Coverage |
|------|-------|---------|
| `vim_handler_tests.rs` | 124 | Comprehensive: all modes (normal/insert/visual/replace), all standard motions, operators (`d`/`c`/`y`/`s`), text objects (words/blocks/quotes/paragraphs), dot-repeat, numbered repeats, registers, case toggle, `f`/`F`/`t`/`T` + `;`/`,`, autosuggestion integration |

### Coverage Summary

| Documented Feature | Tested? | Where |
|--------------------|---------|-------|
| Basic movement (`h/j/k/l`) | Yes | terminal `vim_handler_tests` |
| Word motions (`w/W/b/B/e/E/ge/gE`) | Yes | `word_iterator_tests` + both handler tests |
| Paragraph motions (`{/}`) | Yes | `paragraph_iterator_tests` + `paragraph_tests` |
| Bracket match (`%`) | Yes | `matching_brackets_tests` |
| Operators (`d/c/y/s/x/r`) | Yes | both handler tests |
| Text objects (`iw/aw/i(/a(/i"/a"` etc.) | Yes | `text_objects/*_tests` + both handler tests |
| Character search (`f/F/t/T/;/,`) | Yes | terminal `vim_handler_tests` |
| Mode switching (`i/I/a/A/o/O/v/V`) | Yes | both handler tests |
| Registers (`"a-z`, `"+`, etc.) | Yes | both handler tests |
| Dot-repeat (`.`) | Yes | both handler tests |
| Undo/redo (`u/⌃r`) | Yes | terminal `vim_handler_tests` |
| Case toggle (`~`, `gu`, `gU`) | Yes | terminal `vim_handler_tests` |
| Comment toggle (`gcc/gc`) | Partial | code editor handler tests |
| `/`, `?`, `*`, `#` (Warp search) | Not in vim tests | Likely tested in search/command tests |
