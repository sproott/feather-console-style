# Anti-Patterns

Format: **mistake → why it is wrong → fix**.

## Outdated / Non-existent API

- **Calling `console.Secion "..."`** → the README output table misspells the member; there is no `Secion` and it will not compile → use `console.Section "..."`.

- **Using `console.Indent "text"` as an output call** → the README lists it with the output members, but `Indent` only returns the string prefixed with `Style.Indentation`; nothing is written and the result is silently dropped → print it: `console.Message(console.Indent "text")`.

- **Copying the README `SimpleOptions` sample `[ ["first"; "desc 1"]; ["second", "desc 2"] ]`** → `["second", "desc 2"]` is a list of one tuple, so the outer list does not type-check → separate columns with `;`: `[ "second"; "desc 2" ]`.

- **Relying on the README claim that `Ask` shows its question on `Quiet`** → every styled render, including the `Ask` question, is empty below `Normal`; on `Quiet` only a trailing space is written before `Console.ReadLine()` blocks → don't prompt on `Quiet`, or raise `console.Verbosity` to `Normal` around the `Ask`.

- **Expecting `Error` to print plain red text (as the README table says)** → a one- or two-line error renders as a white-on-red block with a `☠️` prefix on stderr; only an error with more than two lines is plain red text → assert on / design for the block, or use `Message "<c:red>...</c>"` for an inline red line on stdout.

- **Writing `use output = Output.BufferOutput(Verbosity.Normal)` (README "Output" usage)** → `BufferOutput` is `IDisposable`, so omitting `new` triggers warning FS0760 → `use output = new Output.BufferOutput(Verbosity.Normal)`; the same applies to `StreamOutput`.

- **Copying the README custom-tags sample `{ Style.defaults with CustomTags = [ ... ] }`** → replacing the list drops the default `number`, `b`, `i`, `u` tags the README advertises; they then print literally → append: `CustomTags = Style.defaults.CustomTags @ [ ... ]`.

## Markup Pitfalls

- **Nesting markup (`<c:red>a <c:|b>b</c> c</c>`) or custom tags inside each other** → the parser is flat; the inner tag is printed literally and the outer `</c>` leaks into the output → close each `<c:...>` before opening the next, e.g. `<c:red>a </c><c:red|b>b</c><c:red> c</c>`.

- **Spelling modifiers as words (`<c:|bold>`, `<c:|underline>`)** → modifiers are matched by contained letters, so `bold` sets bold **and** dim (`b`, `d`), `underline` sets underline, dim, reverse and italic → use the single letters `b d i u r s`, e.g. `<c:|bu>`.

- **Assuming an invalid color fails loudly** → unknown names (`<c:redd>`) and malformed hex values are dropped silently and the text prints uncolored → use the named palette (`red`, `dark-yellow`, `light-blue`, ...) or `#RRGGBB` / `#RRGGBBAA`, and eyeball new colors once.

- **Rendering or stripping custom tags with `Markup.render` / `Markup.removeMarkup`** → those functions know only `<c:...>`; `<env>`, `<number>`, `<b>` stay in the text → print through `ConsoleStyle` members, and strip with `Style.removeMarkup style` or `console.RemoveMarkup`.

- **Printing untrusted text (user input, file contents, HTML/XML) through `Message` / `Write` / `Table`** → there is no escape syntax; `<c:`, `<b>`, `<i>`, `<u>`, `<number>` or any registered custom tag inside the text are interpreted → write raw text directly to the output, guarded by verbosity: `if console.IsNormal() then console.Output.WriteLine raw`.

- **Passing a format string held in a variable (`console.Message(fmt, x)`) or more than 5 format arguments** → the overloads take a `Printf.StringFormat` literal and stop at 5 arguments; neither compiles → use a literal, an interpolated string (`console.Message $"..."`), or `console.Message(sprintf "..." a b c d e f)`.

## Output and Verbosity

- **Mixing `printfn` / `Console.WriteLine` with `ConsoleStyle` output** → raw prints ignore verbosity, skip markup, don't reach `BufferOutput` in tests or files behind `StreamOutput` → route every human-facing line through the `ConsoleStyle` instance.

- **Looking for `Error` output in `BufferOutput.Fetch()`** → `Error` writes through `WriteErrorLine` (stderr), which `BufferOutput` keeps separately → read it with `FetchError()`.

- **Combining outputs with different verbosity levels (`ConsoleOutput(Verbosity.Normal) <+> BufferOutput(Verbosity.Debug)`)** → `CombinedOutput.combine` throws `"Both outputs must have the same level of verbosity."` → create every combined output with the same `Verbosity.Level`.

- **Relying on the inner outputs' `Verbosity` after setting it on a combined output** → the `CombinedOutput` setter assigns the *previous* level to its inner outputs before storing the new one, so they lag one change behind; a later `<+>` with that inner output can then throw → create outputs at the intended level, or set `Verbosity` on each inner output directly as well.

- **Using `StreamOutput` for files that must keep non-ASCII text** → it encodes with `Encoding.ASCII`, so `✅`, `⚠️`, `☠️`, accented letters and other non-ASCII characters become `?`; with the `(verbosity, stream, errorStream)` constructor, `Dispose` closes only `stream` → for UTF-8 output implement a small `IOutput` over a `StreamWriter`, and dispose a separate error stream yourself.

- **Relying on `console.RemoveMarkup` or `Output.NoMarkup(console)` after changing the style** → both capture the custom tags of the style passed to the constructor; tags added later through `Style <-` / `UpdateStyle` are not stripped (rendering itself does use the new style) → pass the final style to the constructor, or strip with `Style.removeMarkup console.Style`.

- **Expecting `Write` / `WriteLine` to carry the date-time prefix on `VeryVerbose` / `Debug`** → they always render with `NoDateTime`; `Title`, `Section`, `MainTitle`, `List` / `Messages`, option rows, table and tab lines don't get it either → use `Message` (or `Note` / `SubTitle`) for timestamped lines.

- **Wrapping every call in `if not (console.IsQuiet()) then`** → each output member already does nothing below `Normal` → gate only the *extra* detail with `IsVerbose()` / `IsVeryVerbose()` / `IsDebug()`.

## Progress Bars

- **Forgetting `ProgressFinish` (or `use` on child bars)** → the bar is never marked finished nor disposed → always call `console.ProgressFinish progress`; bind children with `use child = console.ProgressStartChild ...` (`Dispose` finishes them).

- **Spawning a child of a child** → a child's `SpawnChild` returns `None`, so `ProgressStartChild child ...` silently returns an inactive no-op bar → keep to one level of children under the main bar.

- **Guarding progress calls with verbosity checks or `null` tests** → `ProgressStart` always returns a `ProgressBar` (inactive below `Normal` or when unavailable) whose members are safe no-ops → use the `Progress*` members unconditionally; plug in other backends via `CreateProgressWith`.
