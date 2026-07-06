---
title: go-yze
---

`go-yze` (package `goyze`) is the **Go analysis runtime** behind gomatic's [`yze`](https://github.com/gomatic/yze) analyzer suite. It registers [`go/analysis`](https://pkg.go.dev/golang.org/x/tools/go/analysis) analyzers, runs them over package patterns through a pluggable driver, and normalizes every finding into a lean, stickler-compatible JSON `Report` — with optional mechanical fixing and post-fix verification.

- **Source:** [`gomatic/go-yze`](https://github.com/gomatic/go-yze)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-yze](https://pkg.go.dev/github.com/gomatic/go-yze)

## Install

```sh
go get github.com/gomatic/go-yze
```

## Concepts

The library is the seam between analyzers (producers of findings) and the [`stickler`](https://github.com/gomatic/stickler) runner (consumer of findings). Its pipeline is: **register → drive → collect → (fix → verify)**.

- **`Registration`** declares one analyzer's identity to the framework: its `*analysis.Analyzer`, a stable `Name`, a documentation `URL`, and semantic `Categories`. Every diagnostic it emits carries the rule id `yze/<name>` (`Registration.RuleID`).
- **`Driver`** is the function type that runs registered analyzers over `[]Pattern` (package patterns like `"./..."`) and returns a shared `*token.FileSet` plus per-analyzer `DriverResult`s. `CheckerDriver` is the default, backed by [`golang.org/x/tools/go/analysis/checker`](https://pkg.go.dev/golang.org/x/tools/go/analysis/checker).
- **`Report`** is the normalized envelope: a slice of `Diagnostic` (tool, rule, path, position, severity, message, and any suggested `Fix`es). Findings in generated files (`// Code generated … DO NOT EDIT.`) are dropped.
- **`Verifier`** reloads the tree after fixes are applied to confirm it still parses and type-checks; `CheckerVerifier` is the default.

## Usage

### Run analyzers and emit a report

`Run` validates registrations, drives them, and returns a normalized `Report`. Consumers supply their own analyzers (the `yze` suite wires in `errconst`, `ptrparam`, and the rest):

```go
package main

import (
	"fmt"
	"os"

	goyze "github.com/gomatic/go-yze"
	"golang.org/x/tools/go/analysis"
)

func main() {
	// myAnalyzer is any *analysis.Analyzer — from the yze suite or your own.
	regs := []goyze.Registration{
		{
			Analyzer:   myAnalyzer,
			Name:       "errconst",
			URL:        "https://gomatic.github.io/docs.yze/errconst/",
			Categories: []goyze.Category{"errors"},
		},
	}

	report, err := goyze.Run(goyze.CheckerDriver, regs, []goyze.Pattern{"./..."})
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	out, err := goyze.MarshalReport(report)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(out))
}

var myAnalyzer *analysis.Analyzer // supplied by the caller
```

`Run` fails fast: an invalid `Registration` returns `ErrMissingName`/`ErrMissingAnalyzer` before the driver runs, a driver failure is wrapped as `ErrDriver`, a load failure as `ErrLoadPackages`, and an analyzer whose `Run` errored as `ErrAnalyzer` — each matchable with `errors.Is`.

### Configure analyzer settings

`ApplyConfig` sets per-analyzer flags before a run. It is keyed by analyzer name, then by flag name; unknown analyzer names are ignored (a config may target a larger suite than is present), an unknown flag is `ErrUnknownSetting`, and a value a known flag rejects is `ErrInvalidSettingValue`:

```go
settings := goyze.Settings{
	"errconst": {"exempt": "go-error"},
}
if err := goyze.ApplyConfig(regs, settings); err != nil {
	// errors.Is(err, goyze.ErrUnknownSetting) — flag not defined
	// errors.Is(err, goyze.ErrInvalidSettingValue) — bad value
}
```

### Apply fixes and verify

When a `Diagnostic` carries `Fixes`, `ApplyFixes` merges every fix's `TextEdit`s per file, rewrites the bytes, reformats, and writes back — deduplicating identical edits and rejecting overlapping ones (`ErrOverlappingEdits`, `ErrEditOutOfBounds`). Reader, writer, and formatter are injected; `GoFormat` is the gofmt default:

```go
var fixes []goyze.Fix
for _, d := range report.Diagnostics {
	fixes = append(fixes, d.Fixes...)
}

result, err := goyze.ApplyFixes(os.ReadFile, writeFile, goyze.GoFormat, fixes)
if err != nil {
	// errors.Is(err, goyze.ErrReadFile | ErrFormat | ErrWriteFile)
}
fmt.Printf("changed %d files, applied %d edits\n", result.FilesChanged, result.EditsApplied)

// Confirm the fixed tree still compiles (test files included).
v, err := goyze.CheckerVerifier([]goyze.Pattern{"./..."})
if err == nil && !v.Clean() {
	for _, issue := range v.Issues {
		fmt.Fprintln(os.Stderr, issue) // "file:line:col: message"
	}
}

func writeFile(path string, data []byte) error { return os.WriteFile(path, data, 0o644) }
```

`ApplyEdits` is the pure, in-memory core underneath `ApplyFixes`: given content and a set of byte-range `TextEdit`s (in any order), it returns the rewritten bytes without touching disk, so edit application is testable in isolation.

## Design

- **Everything is an injected seam.** `Driver`, `Verifier`, and the `FileReader`/`FileWriter`/`Formatter` triple are function types, so the loader, filesystem, and formatter are all substitutable — the `Checker*` defaults are the only pieces that touch real packages or disk.
- **Fail loud, never a false pass.** The loader gate (`ErrLoadPackages`) rejects a run that matched no packages or loaded any package with parse/type errors (with a `go.work` workspace hint), and `ErrAnalyzer` drills a failed action down to the dependency whose own `Run` produced the error — because the `go/analysis` checker silently skips errored packages, which would otherwise degrade a broken run to zero diagnostics and a clean exit.
- **Immutable, value-typed contracts.** `Diagnostic`, `Fix`, `TextEdit`, `Report`, and the domain newtypes (`Pattern`, `AnalyzerName`, `Category`, `Severity`, …) are plain values, safe to copy and compare; `ApplyEdits` never mutates its input.
- **Errors are constants.** Every failure the package emits is an [`errs.Const`](https://pkg.go.dev/github.com/gomatic/go-error#Const) sentinel matched with `errors.Is`, never by string — wrapping a cause through `.With` keeps both the sentinel and the cause recoverable.
