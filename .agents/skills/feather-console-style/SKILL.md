---
name: feather-console-style
description: Use whenever generating or reviewing F# code that styles console/CLI output with the Feather.ConsoleStyle library — creating a `ConsoleStyle` and calling `Title`, `Section`, `SubTitle`, `MainTitle`, `Message`, `Write`/`WriteLine`, `Success`, `Warning`, `Error`, `Note`, `List`, `Messages`, `Options`/`SimpleOptions`/`GroupedOptions`, `Table`, `Tabs`, `Ask`, or the progress-bar members (`ProgressStart`, `ProgressStartChild`, `ProgressAdvance`, `ProgressFinish`, `CreateProgressWith`). Trigger also on `<c:COLOR|bg:COLOR|bdiurs>` markup, `Markup.render`/`Markup.removeMarkup`, `Style.defaults`, `CustomTag`, `TagName`, `MarkupString`, `ShowDateTime`, `Verbosity.Level` (`Quiet`..`Debug`), `IOutput` implementations (`ConsoleOutput`, `PrintOutput`, `BufferOutput`, `StreamOutput`, `NoMarkup`, `CombinedOutput` / `<+>`), or testing console output via `BufferOutput.Fetch`.
---

# Feather-Console-Style

Library: [FeatherTools/console-style](https://github.com/FeatherTools/console-style)
NuGet: `Feather.ConsoleStyle`

## Purpose

`Feather.ConsoleStyle` is an F# library (inspired by SymfonyStyle) for styled console I/O in command-line tools. A single `ConsoleStyle` object renders titles, sections, message blocks, option lists, tables, colored tabs, figlet banners and progress bars, interprets an inline `<c:...>` color/format markup (rendered as 24-bit ANSI escape codes), gates output by a verbosity level, and writes through a pluggable `IOutput` (terminal, `printf`, in-memory buffer, stream, markup-stripping wrapper, or a combination).

## When to Use

- Building a CLI/console tool that needs consistent titles, sections, success/warning/error blocks, lists, options help and tables.
- Coloring parts of a message inline with `<c:...>` markup or reusable custom tags.
- Supporting `-q` / `-v` / `-vv` / `-vvv` style verbosity with one switch.
- Showing (nested) progress bars for long-running work.
- Capturing styled output in tests via `BufferOutput`, or mirroring it to a file without ANSI codes.

## When NOT to Use

- Structured/application logging (levels, sinks, JSON) — this library only formats human-facing console text.
- Rich interactive TUIs (menus, prompts with validation, live layouts) — input is limited to `Ask` (a raw `Console.ReadLine`).
- Rendering untrusted text verbatim — there is no escaping, so any `<c:` or registered custom tag in the text is interpreted.

## Main Concepts

- **ConsoleStyle** — the entry point class. Constructors: `ConsoleStyle()`, `ConsoleStyle(verbosity)`, `ConsoleStyle(style)`, `ConsoleStyle(output)`, `ConsoleStyle(output, style)`. Exposes the output members, `Verbosity` (get/set), `Is*()` checks, `Output` / `Style` (get/set), `UpdateStyle`, `Indent` (returns a string) and `RemoveMarkup`.
- **Output members** — `Write`, `WriteLine` (never prefix a date-time), `Message`, `Title`, `SubTitle`, `Section`, `Note`, `Success`, `Warning`, `Error` (to stderr), `NewLine`, `MainTitle` (figlet banner); each text member has `(format, a)` .. `(format, a, b, c, d, e)` overloads that call `sprintf`.
- **Output-many members** — `List`, `Messages prefix`, `Options`, `SimpleOptions`, `GroupedOptions separator`, `Table header rows`, `Tabs(tabs)` / `Tabs(tabs, length)`.
- **Markup** — `<c:FOREGROUND|bg:BACKGROUND|MODIFIERS>text</c>`; colors are named (normalized: case, `-`, `_` ignored) or `#RRGGBB` / `#RRGGBBAA`; modifiers are single letters `b` `d` `i` `u` `r` `s`. `Markup` record + module (`empty`, `parse`, `asString`, `render`, `removeMarkup`, `hasMarkup`).
- **Style** — record (`ShowDateTime`, `MainTitleUnderline`, `TitleUnderline`, `SectionUnderline`, `Indentation`, `BlockLength`, `CustomTags`); start from `Style.defaults`. `Style.removeMarkup style` strips custom tags and markup.
- **CustomTag** — `{ Tag: TagName; Markup: MarkupString }` mapping `<tag>..</tag>` to markup; built with `CustomTag.createWithMarkup`, `CustomTag.createAndParseMarkup` (returns `Result<CustomTag, string>`) or a record literal with `MarkupString.create`. Defaults: `number`, `u`, `i`, `b`.
- **ShowDateTime** — `NoDateTime` | `ShowDateTimeAs of format` | `ShowDateTimeFor of Map<Verbosity.Level, string>` (default: time on `VeryVerbose`, date-time on `Debug`).
- **Verbosity.Level** — `Quiet` | `Normal` | `Verbose` | `VeryVerbose` | `Debug`; every styled output requires at least `Normal`.
- **IOutput** (namespace `Feather.ConsoleStyle.Output`) — `ConsoleOutput`, `PrintOutput`, `BufferOutput` (`Fetch` / `FetchError`), `StreamOutput`, `NoMarkup`, and `CombinedOutput.combine` / `CombinedOutput.Operators.(<+>)`.
- **Tab** — `{ Text; Value: string option; Color: System.Drawing.Color }`; `Tab.parseColor`, `Tab.ofString`, `Tab.create`.
- **ProgressBar** / **IProgress** — `ProgressStart` returns a `ProgressBar` (inactive no-op below `Normal`); one child level via `ProgressStartChild` / `ProgressStartChildAndKeepIt`; custom backends via `CreateProgressWith`.

## Related Libraries

- **ShellProgressBar** — backend of the default progress bar.
- **Colorful.Console** — supplies `Colorful.Figlet` / `Colorful.FigletFont` for `MainTitle` overloads.

## Keywords for Search

Feather.ConsoleStyle, console-style, ConsoleStyle, SymfonyStyle, CLI output, console colors, ANSI, Title, SubTitle, Section, MainTitle, figlet, Message, Write, WriteLine, Success, Warning, Error, Note, List, Messages, Options, SimpleOptions, GroupedOptions, Table, Tabs, Tab, Ask, markup, `<c:>`, Markup.render, Markup.removeMarkup, Style, Style.defaults, CustomTag, TagName, MarkupString, ShowDateTime, Indentation, Underline, BlockLength, Verbosity, Quiet, Verbose, VeryVerbose, Debug, IOutput, ConsoleOutput, PrintOutput, BufferOutput, StreamOutput, NoMarkup, CombinedOutput, `<+>`, ProgressBar, IProgress, ProgressStart, ProgressStartChild, ProgressAdvance, ProgressFinish, CreateProgressWith, F#

## Reference Files

- For composition principles, markup rules, verbosity behaviour, outputs, and testing guidance, read `references/preferred-patterns.md`.
- For known pitfalls, README inaccuracies, and library quirks, read `references/anti-patterns.md`.
- For worked, self-contained code examples ordered by increasing complexity, read `references/examples.md`.
