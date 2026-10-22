# repl

A line editor and a read-eval-print loop for [sysl](https://github.com/sysl-lang/sysl) that work the
same at a macOS or Linux terminal and over a serial cable to a board. A program that needs a REPL uses
this one and is portable from the start.

```
sh/sysl/repl/
    terminal.sysl       the Terminal trait: five members, none of which knows what a key is
    keys.sysl           escape sequences and UTF-8 decoded into keys, once, for every terminal
    line.sysl           the line: UTF-8 in caller-owned bytes, a cursor that never splits a character
    history.sysl        the history: packed into a fixed number of bytes, oldest dropped first
    editor.sysl         reading a line: bindings, completion, hints, horizontal scrolling
    repl.sysl           the loop: read, evaluate, print, continuation lines, Ctrl-C and Ctrl-D
    mock.sysl           a scripted terminal, for tests: bytes in, exact bytes out
    host/__posix__/     the Unix terminal, history files, and an editor with allocated storage
examples/
    echo.sysl           an echo REPL at your terminal, with completion, hints and a history file
    board.sysl          the same loop over a UART, built for a board with no heap
package.hocon           who this package is
```

There are two modules. **`sh.sysl.repl`** is the core, and it is `@no_alloc` with no capability
requirement, so it runs on a board with no allocator and no operating system. **`sh.sysl.repl.host`**
is the Unix part. Its files are under `__posix__`, so a freestanding build never sees them.

## Using it

Add it to your project's `package.hocon` and `sysl build` fetches it:

```
dependencies {
  repl { git = "github.com/sysl-lang/repl", version = "0.1.0" }
}
```

A whole REPL at the terminal looks like this:

```
import sh.sysl.repl.{Done, Quit}
import sh.sysl.repl.host.{config, repl}

repl(config("calc> "), (line, out) ->
    if line == "bye" then return Quit

    out.print(f"you said: $line")
    Done)
```

The evaluator gets the whole input as a `string` and an `Out` to write to. It returns one of three
outcomes:

- **`Done`** records the input in the history and shows the prompt again.
- **`More`** says the input isn't finished, for example an open bracket or a definition still going.
  The loop shows the continuation prompt and joins the next line on with a newline.
- **`Quit`** ends the loop.

Ctrl-C abandons the input, including any continuation, and Ctrl-D on an empty line ends the loop. A
`Config` also takes a banner, a continuation prompt and a history file (`home_file(".calc_history")`).
`repl_with` adds completion and hints.

To try the example, run this from the repository root:

```
sysl run examples/echo.sysl
```

## On a board

The core needs a terminal and storage, and nothing else. A UART terminal takes a few lines:

```
struct Uart
end Uart

impl Terminal for Uart
    read_byte(*self) -> Option[u8]
        val c = uart_getc()

        if c < 0 then None else Some(u8(c))

    write(*self, bytes: []const u8) -> bool
        for b in bytes do uart_putc(int(b))

        true

    flush(*self) -> bool = true
    enter_raw_mode(*self) -> bool = true
    exit_raw_mode(*self) = ()
```

The storage is module arrays:

```
var line_store: [128]u8 = [0; 128]
var draft_store: [128]u8 = [0; 128]
var history_store: [1024]u8 = [0; 1024]
var join_store: [512]u8 = [0; 512]

var ed = editor(line_store, draft_store, history_store, 16)
var uart = Uart()

run(&ed, &uart, join_store, prompts("pico> "), (line, out) -> ...)
```

`examples/board.sysl` is exactly this. `sysl build-c examples/board.sysl --target thumb-freestanding`
builds it into an archive whose only undefined symbols are `uart_getc`, `uart_putc`, `memcpy` and
`memset`.

**A Pico 2 UART and USB-CDC terminal is next.** It's a terminal of the shape above, so it adds no
code to the core.

## The keys

| key | does |
|---|---|
| Left, Right (or Ctrl-B, Ctrl-F) | move one character |
| Ctrl-Left, Ctrl-Right (Alt-Left/Right, Alt-b, Alt-f) | move one word |
| Home, End (or Ctrl-A, Ctrl-E) | go to the start or end of the line |
| Backspace, Delete | delete one character |
| Alt-Backspace, Ctrl-W | delete the word before the cursor |
| Ctrl-Delete, Alt-Delete, Alt-d | delete the word after the cursor |
| Ctrl-K, Ctrl-U | delete to the end or start of the line |
| Up, Down (or Ctrl-P, Ctrl-N) | walk the history; Down past the newest restores what you were typing |
| Tab | complete (see below) |
| Ctrl-L | clear the screen |
| Ctrl-C | abandon the line |
| Ctrl-D | end of input on an empty line, Delete otherwise |

**A symbol is a word of its own.** In `x+=y-1` the word keys stop at `x`, `+=`, `y`, `-` and `1`,
which is what you want when moving through an expression. Every character beyond ASCII counts as a
word character.

**The history skips blank lines and repeats of the newest entry**, after trimming whitespace from
both ends. It's packed into a fixed number of bytes, and the oldest entries go when a new one would
not fit or when the entry limit is reached.

**Editing is by code point.** The cursor moves and deletes whole characters, so it can never split a
multibyte one. Bytes that aren't UTF-8 arrive as U+FFFD. Wide characters take two columns.

**A line wider than the terminal scrolls sideways**, keeping the cursor in view. The width comes from
the terminal (`TIOCGWINSZ` on Unix). A terminal that can't report it, such as a serial line, uses the
editor's `width`, which defaults to 80.

## Completion and hints

Both are closures, with nothing stored and nothing allocated:

- `complete(word, i)` returns the `i`th candidate for the word before the cursor (the run of non-blank
  characters ending there), or `None` when there are no more.
- `hint(line)` returns text to show dimmed after the cursor while the cursor is at the end of the
  line. A hint is never inserted.

**What Tab does:**

- With no candidates, it rings the bell.
- With one candidate, it replaces the word with it.
- With several, it extends the word to their longest common prefix. When the word can't grow any
  further, it lists the candidates under the line and redraws the line below them.

This is how bash behaves. Cycling through candidates was the alternative, but it hides the choices
from you.

## The Terminal trait

```
trait Terminal
    read_byte(*self) -> Option[u8]
    write(*self, bytes: []const u8) -> bool
    flush(*self) -> bool
    enter_raw_mode(*self) -> bool
    exit_raw_mode(*self)
    columns(*self) -> usize = 0
```

Key decoding and every escape sequence the editor writes live in the core, so a terminal only moves
bytes. `columns` is the one optional member. Raw mode is entered for each line and left after it, so
the evaluator runs with the terminal in its ordinary mode and Ctrl-C there still interrupts the
program on a host. On a board, `\n` in what the evaluator prints goes out as `\r\n`, and the editor
always writes CRLF.

## Testing your own REPL

`MockTerminal` is public, so you can test your own REPL the same way this package is tested. It is
built from a script of input bytes and storage for output, and allocates nothing:

```
var out: [4096]u8 = [0; 4096]
var t = mock_terminal("1 2 +\r".bytes, out)

run(&ed, &t, join, prompts("> "), eval)
assert(t.text().contains("3\r\n"))
```

It can also make writes fail after a given count (`writes_left`) and report a width (`width`).

## What it doesn't do yet

- **The Pico 2 terminals.** The core is ready for them; they come next.
- **Windows.** A console there is a different API, not a different byte stream.
- **Async.** The loop blocks on `read_byte`. An async variant would duplicate the loop rather than
  change the core, and nothing in the org needs one yet.

## License

[ISC](LICENSE)
