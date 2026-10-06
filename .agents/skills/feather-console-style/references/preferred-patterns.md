# Preferred Patterns

## Core Principles

- Create one `ConsoleStyle` per command run and pass it to the functions that print. Depend on `ConsoleStyle` rather than `printfn` / `Console.Write`, so verbosity, style and output target stay switchable in one place.
- Express intent through the semantic members (`Title`, `Section`, `Success`, `Warning`, `Error`, `Note`, `Table`, `Options`) and inline markup. Don't hand-build underlines, padding or ANSI escape codes.
- Choose the output target (`IOutput`) at the composition root; the printing code never knows whether it writes to a terminal, a buffer or a file.

## Recommended API Usage

- Headings: `Title` (underlined with `=`, blank line after), `Section` (underlined with `-`, blank line after), `SubTitle` (yellow + underlined, no blank line), `MainTitle` (figlet banner, markup stripped from the text). See `examples.md` → Basic output.
- Plain text: `Message` (line, may get a date-time prefix), `WriteLine` / `Write` (never a date-time prefix; `Write` adds no newline).
- Blocks: `Success` and `Warning` render a padded block at least `Style.BlockLength` wide; `Error` renders a block sized to the message and writes to the **error** stream. An `Error` with more than two lines (more than one `\n`) renders as plain red text, not a block.
- Formatting: use the `(format, a ..)` overloads up to 5 arguments (`console.Message("Found %d in %s", 3, "registry")`); beyond that, pass `sprintf ...` as a single string.
- Lists and help screens: `List` (` - ` prefix), `Messages prefix lines`, `Options` (`-` before each first column), `SimpleOptions` (no prefix), `GroupedOptions separator` (groups rows by the text before `separator` in the first column, sorted). Each inner `string list` is one row of aligned columns. See `examples.md` → Tables and options.
- Tables: `Table header rows` with `string list` / `string list list`; header markup is stripped and rendered dark-yellow, row markup is kept and ignored when aligning.
- Tabs: build `Tab` values with `Tab.parseColor colorName text` (unknown color → white) and set `Value = Some ...` for a second line; `Tabs(tabs, length)` fixes the tab width.
- Input: `Ask question` writes the question styled as a subtitle and returns `Console.ReadLine()` (`null` at end of input).

## Markup Rules

- Shape: `<c:FOREGROUND|bg:BACKGROUND|MODIFIERS>text</c>`; every part is optional (`<c:red>`, `<c:|bg:blue>`, `<c:|bu>`, `<c:>`).
- Colors: named (`red`, `dark-yellow`, `light_blue`, ... — case, `-` and `_` are ignored) or hex `#RRGGBB` / `#RRGGBBAA`. Unknown names are silently ignored.
- Modifiers are single letters: `b` bold, `d` dim, `i` italic, `u` underline, `r` reverse, `s` strike-through.
- Tags do not nest; close one `<c:...>` before opening the next.
- Custom tags (`<env>prod</env>`) are plain text replacements into `<c:...>` markup, applied by `ConsoleStyle` and `Style.removeMarkup`; `Markup.render` / `Markup.removeMarkup` know only `<c:...>`.
- `Markup.render` returns the ANSI string, `Markup.removeMarkup` the plain string (also strips already-rendered ANSI codes); `Markup.parse` / `Markup.asString` convert between the `":fg|bg:bg|mods"` string and the `Markup` record.

## Error Handling

- The library has no error channel for output; malformed markup degrades to literal text and unknown colors are dropped.
- `CustomTag.createAndParseMarkup` returns `Result<CustomTag, string>` (`"Empty tag"` / `"Empty markup"`); match on it at startup instead of throwing. See `examples.md` → Custom style and tags.
- `CombinedOutput.combine` / `<+>` throws when the two outputs have different verbosity levels — create all combined outputs with the same `Verbosity.Level`.

## Composition

- Customize with a record copy of `Style.defaults`. Append to `Style.defaults.CustomTags` (`Style.defaults.CustomTags @ [ ... ]`) to keep the built-in `number`, `b`, `i`, `u` tags.
- Change style at runtime with `console.Style <- style` or `console.UpdateStyle(fun style -> ...)`.
- Compose outputs: `Output.ConsoleOutput(v) <+> Output.NoMarkup(Style.defaults, fileOutput)` writes colored text to the terminal and plain text to a file. `console.Output <- Output.NoMarkup(console)` turns off colors for an existing console (e.g. `--no-ansi`). See `examples.md` → Output targets.
- Progress: `ProgressStart` → `ProgressAdvance` per step → `ProgressFinish`; children via `ProgressStartChild parent message total` (collapsed when done) or `ProgressStartChildAndKeepIt`, bound with `use`. Only one level of children is supported. Swap the backend with `CreateProgressWith(fun message -> ... :> IProgress)`; `ProgressStartDefault(message, total)` always uses the built-in bar. See `examples.md` → Progress bars.

## Verbosity

- Levels are cumulative: `IsNormal()` is true for `Normal`..`Debug`, `IsVerbose()` for `Verbose`..`Debug`, `IsVeryVerbose()` for `VeryVerbose` and `Debug`, `IsDebug()` only for `Debug`; `IsQuiet()` only for `Quiet`.
- On `Quiet`, every output member is a no-op, progress bars are inactive, and `Ask` prints only a trailing space before reading input.
- Gate extra detail explicitly: `if console.IsVerbose() then console.Message ...`. See `examples.md` → Verbosity.
- With `Style.defaults`, `Message`, `SubTitle`, `Note`, the `Options` / `SimpleOptions` titles, `Ask` and the `Success` / `Warning` / `Error` output get a `[HH:mm:ss]` prefix on `VeryVerbose` and `[yyyy-MM-dd HH:mm:ss]` on `Debug`; only the first line of a multi-line message is prefixed. Set `ShowDateTime = NoDateTime` to disable it.

## Integration with Other Libraries

- `MainTitle(title, font: Colorful.FigletFont)` and `MainTitle(title, figlet: Colorful.Figlet)` take Colorful.Console types; load fonts with `Colorful.FigletFont.Load "file.flf"`.
- The default progress bar is ShellProgressBar, which redraws the terminal in place; prefer a custom `IProgress` (via `CreateProgressWith`) for CI logs.

## Naming Conventions

- Output types live in `Feather.ConsoleStyle.Output`; with `open Feather.ConsoleStyle` refer to them as `Output.BufferOutput`, `Output.NoMarkup`, etc., and `open Feather.ConsoleStyle.Output.CombinedOutput.Operators` for `<+>`.
- `BufferOutput` and `StreamOutput` are `IDisposable`: create them with `new` and bind with `use`.
- Name custom tags by meaning (`env`, `path`, `danger`), not by color.

## Testing Recommendations

- Construct `ConsoleStyle(new Output.BufferOutput(level))` (optionally with a style), run the code, then assert on `Fetch()` (stdout) and `FetchError()` (stderr); both clear the buffer. See `examples.md` → Testing with BufferOutput.
- Strip ANSI and custom tags with `console.RemoveMarkup` (or `Style.removeMarkup style`) before comparing text, or assert on the exact escape sequences when color matters.
- Keep `ShowDateTime = NoDateTime` (or test at `Normal`/`Verbose`) when asserting exact lines; otherwise normalize the time prefix.
- Cover the `Quiet` case: nothing should be written to either buffer.
