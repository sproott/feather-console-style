# Examples

All example code for this skill lives here. Examples are ordered by increasing complexity and each is self-contained. Neutral placeholder names are used throughout. Output blocks show the text with markup/ANSI codes removed.

## Basic output

```fs
open Feather.ConsoleStyle

let console = ConsoleStyle()

console.Title "Deploy"
console.Section "Prerequisites"
console.Message "Checking <c:cyan>docker</c> ..."
console.Message("Found <number>%d</number> images in <c:yellow>%s</c>", 3, "registry")
console.SubTitle "Summary"
console.List [ "api"; "worker" ]
console.Note "Dry run - nothing was pushed"
console.Success "Done"
console.Error "Push failed"     // stderr
```

Output (`Success` block padded to `Style.BlockLength` = 120, shortened here):

```
Deploy
======

Prerequisites
-------------

Checking docker ...
Found 3 images in registry
Summary
 - api
 - worker
 ! [NOTE] Dry run - nothing was pushed

                         ...
 ✅ Done                 ...
                         ...

```

## Inline markup

```fs
open Feather.ConsoleStyle

let console = ConsoleStyle()

console.Message "Plain, <c:green>green</c>, <c:|b>bold</c>, <c:white|bg:dark-red|bu>white on red, bold + underlined</c>"
console.Message "Hex colors: <c:#D20000>#D20000</c> and <c:black|bg:#43ff64d9>RGBA background</c>"
console.Message "Default tags: <number>42</number>, <b>bold</b>, <i>italic</i>, <u>underline</u>"

console.Write "no newline, "
console.WriteLine "end of line"

let rendered = "Status: <c:green>ok</c>" |> Markup.render        // "Status: \u001b[38;2;50;205;50mok\u001b[0m"
let plain = "Status: <c:green>ok</c>" |> Markup.removeMarkup     // "Status: ok"
```

## Custom style and tags

```fs
open Feather.ConsoleStyle

let style =
    { Style.defaults with
        Indentation = Indentation "  "
        TitleUnderline = Underline "#"
        BlockLength = 60
        ShowDateTime = NoDateTime
        CustomTags =
            Style.defaults.CustomTags @ [
                CustomTag.createWithMarkup (TagName "env") { Markup.empty with Foreground = Some "dark-cyan"; Bold = true }
                { Tag = TagName "path"; Markup = MarkupString.create ":gray|i" }
            ]
    }

let console = ConsoleStyle(style)

console.Title "Release"
console.Message "Deploying to <env>prod</env> from <path>./dist</path>"

match CustomTag.createAndParseMarkup (TagName "danger") ":white|bg:red|b" with
| Ok tag -> console.UpdateStyle(fun style -> { style with CustomTags = style.CustomTags @ [ tag ] })
| Error message -> console.Error("Invalid custom tag: %s", message)

console.Message "This is <danger>irreversible</danger>"
```

Output:

```
Release
#######

Deploying to prod from ./dist
This is irreversible
```

## Tables and options

```fs
open Feather.ConsoleStyle

let console = ConsoleStyle()

console.Table [ "Service"; "Status"; "Replicas" ] [
    [ "api"; "<c:green>running</c>"; "3" ]
    [ "worker"; "<c:red>failed</c>"; "0" ]
]

console.Options "Options:" [
    [ "-v, --verbose"; "Increase verbosity" ]
    [ "-q, --quiet"; "Do not output any message" ]
]

console.SimpleOptions "Arguments:" [
    [ "environment"; "Target environment"; "<c:yellow>[default: \"dev\"]</c>" ]
]

console.GroupedOptions ":" "Available commands:" [
    [ "list"; "Lists commands" ]
    [ "deploy:release"; "Release a package" ]
    [ "deploy:rollback"; "Roll back a release" ]
    [ "cache:clear"; "Clear the cache" ]
]

[ "green", "api", "up"; "red", "worker", "down" ]
|> List.map (fun (color, name, status) -> { Tab.parseColor color name with Value = Some status })
|> console.Tabs
```

Output (trailing spaces trimmed; tabs are colored backgrounds):

```
--------- --------- ----------
 Service   Status    Replicas
--------- --------- ----------
 api       running   3
 worker    failed    0
--------- --------- ----------

Options:
    - -v, --verbose  Increase verbosity
    - -q, --quiet    Do not output any message

Arguments:
    environment  Target environment  [default: "dev"]

Available commands:
    list             Lists commands
 cache
    cache:clear      Clear the cache
 deploy
    deploy:release   Release a package
    deploy:rollback  Roll back a release


   api        worker
    up         down

```

## Verbosity

```fs
open Feather.ConsoleStyle

let verbosityOf (args: string list) =
    if args |> List.contains "-q" then Verbosity.Quiet
    elif args |> List.contains "-vvv" then Verbosity.Debug
    elif args |> List.contains "-vv" then Verbosity.VeryVerbose
    elif args |> List.contains "-v" then Verbosity.Verbose
    else Verbosity.Normal

let run args =
    let console = ConsoleStyle(verbosityOf args)

    console.Message "Always shown unless quiet"

    if console.IsVerbose() then
        console.Message "Resolved config from <c:gray>./config.yaml</c>"

    if console.IsDebug() then
        console.Message("Raw args: %A", args)

    console.Verbosity <- Verbosity.Quiet
    console.Message "never printed"
```

Output of `run [ "-vvv" ]` (date-time prefix comes from `Style.defaults`):

```
[2026-01-31 12:00:00]    Always shown unless quiet
[2026-01-31 12:00:00]    Resolved config from ./config.yaml
[2026-01-31 12:00:00]    Raw args: ["-vvv"]
```

## Testing with BufferOutput

```fs
module Greeting.Tests

open Expecto
open Feather.ConsoleStyle

let greet (console: ConsoleStyle) name =
    console.Success("Hello %s", name)

[<Tests>]
let tests =
    testList "greet" [
        testCase "should print a success block with the name" <| fun _ ->
            use output = new Output.BufferOutput(Verbosity.Normal)
            let console = ConsoleStyle(output)

            greet console "Jon"

            let printed = output.Fetch() |> console.RemoveMarkup
            Expect.stringContains printed "Hello Jon" "success message should contain the name"

        testCase "should write errors to the error buffer" <| fun _ ->
            use output = new Output.BufferOutput(Verbosity.Normal)
            let console = ConsoleStyle(output)

            console.Error "boom"

            Expect.equal (output.Fetch()) "" "stdout buffer should stay empty"
            Expect.stringContains (output.FetchError() |> console.RemoveMarkup) "boom" "stderr buffer should contain the error"

        testCase "should print nothing when quiet" <| fun _ ->
            use output = new Output.BufferOutput(Verbosity.Quiet)
            let console = ConsoleStyle(output)

            greet console "Jon"

            Expect.equal (output.Fetch()) "" "quiet console should not write"
    ]
```

Asserting exact colors: the library's own tests compare raw ANSI, e.g. `console.Write("Jon <c:%s>%s</c>", "cyan", "Snow")` buffers `"Jon \u001b[38;2;0;255;255mSnow\u001b[0m"`.

## Output targets

Colored on the terminal, plain text in a log file. `StreamOutput` writes ASCII, so non-ASCII characters become `?` in the file.

```fs
open System.IO
open Feather.ConsoleStyle
open Feather.ConsoleStyle.Output.CombinedOutput.Operators

let run () =
    let verbosity = Verbosity.Normal

    use logFile = new FileStream("run.log", FileMode.Create)
    use fileOutput = new Output.StreamOutput(verbosity, logFile)

    let output = Output.ConsoleOutput(verbosity) <+> Output.NoMarkup(Style.defaults, fileOutput)
    let console = ConsoleStyle(output)

    console.Message "Shown in <c:green>color</c> on the terminal, plain in run.log"

/// Disables colors on an existing console, e.g. for a `--no-ansi` flag.
let plainForCi (console: ConsoleStyle) =
    console.Output <- Output.NoMarkup(console)
```

## Progress bars

One child level under the main bar; children collapse when finished (`ProgressStartChildAndKeepIt` keeps them).

```fs
open Feather.ConsoleStyle

let import (console: ConsoleStyle) (batches: int list list) =
    let progress = console.ProgressStart "Importing" batches.Length

    for batch in batches do
        use child = console.ProgressStartChild progress "Batch" batch.Length

        for _ in batch do
            console.ProgressAdvance child

        console.ProgressAdvance progress

    console.ProgressFinish progress
```

## Custom progress backend

Replace the ShellProgressBar backend, e.g. with log lines for CI. `SpawnChild` returning `None` makes `ProgressStartChild` return an inactive bar.

```fs
open System
open Feather.ConsoleStyle

type LogProgress(console: ConsoleStyle, name: string) =
    let mutable total = 0
    let mutable current = 0

    interface IProgress with
        member _.Start(t) = total <- t
        member _.Advance() =
            current <- current + 1
            console.Message("%s: %d/%d", name, current, total)
        member _.Finish() = console.Message("%s: finished", name)
        member _.SpawnChild(_, _) = None
        member _.IsAvailable() = true

    interface IDisposable with
        member this.Dispose() = (this :> IProgress).Finish()

let useLogProgress (console: ConsoleStyle) =
    console.CreateProgressWith(fun name -> new LogProgress(console, name) :> IProgress)
```

Output of `import` with `useLogProgress` and `[ [ 1; 2 ]; [ 3 ] ]`:

```
Importing: 1/2
Importing: 2/2
Importing: finished

```

## Full workflow — CLI entry point

```fs
open Feather.ConsoleStyle

let style =
    { Style.defaults with
        CustomTags = Style.defaults.CustomTags @ [ { Tag = TagName "service"; Markup = MarkupString.create ":#0b6695|u" } ] }

let verbosityOf (argv: string array) =
    if argv |> Array.contains "-q" then Verbosity.Quiet
    elif argv |> Array.contains "-v" then Verbosity.Verbose
    else Verbosity.Normal

[<EntryPoint>]
let main argv =
    let console = ConsoleStyle(Output.ConsoleOutput(verbosityOf argv), style)

    console.MainTitle "deployer"
    // or with a custom figlet font: console.MainTitle("deployer", Colorful.FigletFont.Load "chunky.flf")

    match console.Ask "Which service?" with
    | null | "" ->
        console.Error "No service given"
        1
    | service ->
        console.Section("Deploying <service>%s</service>", service)

        if console.IsVerbose() then
            console.Table [ "Key"; "Value" ] [
                [ "service"; service ]
                [ "verbosity"; string console.Verbosity ]
            ]

        console.Success("Deployed <b>%s</b>", service)
        0
```
