# go-dictionary

A small command-line dictionary written in Go. You can add words with their definitions, look a word up, list everything alphabetically, and remove words. The data is saved on disk in an embedded [BadgerDB](https://github.com/dgraph-io/badger) key-value store, so it is still there the next time you run the program.

## Origin

I wrote this while following the Udemy course **[Le langage Go : formation complète](https://www.udemy.com/course/le-langage-go-formation-complete/learn/)**, a French-language course on Go. This project is the course's third exercise (`03-dictionary`) and is part of my Go learning.

## Features

| Action   | Description                                                       |
| -------- | ----------------------------------------------------------------- |
| `list`   | Prints every entry, sorted alphabetically (the default action)    |
| `add`    | Adds a word with its definition, or overwrites an existing one    |
| `define` | Prints the definition of a single word                            |
| `remove` | Deletes a word from the dictionary                                |

## Getting started

### Requirements

- Go 1.21+ (tested with Go 1.26)

### Build

```bash
git clone git@github.com:tonystrawberry/go-dictionary.git
cd go-dictionary
go build -o dict .
```

### Usage

The action is chosen with the `-action` flag. Any other arguments come after the flags.

```bash
# Add words (put multi-word definitions in quotes)
./dict -action add golang "A programming language by Google"
./dict -action add badger "A fast key-value store"

# List all entries (same as running ./dict with no flags)
./dict -action list
# Dictionary Content
# Badger      A fast key-value store               Oct 24 17:27:24
# Golang      A programming language by Google     Oct 24 17:27:24

# Look up one word
./dict -action define golang

# Remove a word
./dict -action remove golang
```

The database is created in a `./badger` directory next to where you run the program (it is listed in `.gitignore`). Delete that folder to start over with an empty dictionary.

> BadgerDB prints its own `INFO`/`DEBUG` log lines to stderr. They are harmless. Add `2>/dev/null` to the command to hide them.

## Project structure

```
.
├── main.go                  # CLI entry point: flag parsing and dispatch to actions
├── dictionary/
│   ├── dictionary.go        # Dictionary and Entry types, opening and closing the DB
│   └── actions.go           # Add / Get / Remove / List, gob encoding helpers
├── go.mod
└── go.sum
```

## How it works

### `main.go`: the CLI

- Uses the standard `flag` package to read `-action` (default: `list`). Positional arguments come from `flag.Args()`.
- Opens the dictionary with `dictionary.New("./badger")` and uses `defer d.Close()` so the database is closed properly when the program exits.
- A `switch` on the action calls `actionList`, `actionAdd`, `actionDefine` or `actionRemove`.
- `handleErr` prints any error and exits with status code `1`.

### `dictionary/dictionary.go`: the types

- `Dictionary` wraps a `*badger.DB`.
- `Entry` holds `Word`, `Definition` and `CreatedAt time.Time`.
- `Entry` implements the `fmt.Stringer` interface (`String() string`), so `fmt.Println(entry)` prints a neatly padded row using `%-10v` / `%-50v` widths and a `time.Stamp` date.
- `New(dir)` opens (or creates) the Badger database in `dir`.

### `dictionary/actions.go`: storage logic

- **Serialization:** each `Entry` is encoded to bytes with `encoding/gob` before it is stored, and decoded again when it is read. Badger only stores raw `[]byte` keys and values, so this step is needed.
- **Add:** capitalizes the displayed word with `strings.Title`, records `time.Now()`, and writes it in a read-write transaction (`db.Update` + `txn.Set`). The key is the word exactly as you typed it.
- **Get:** reads one key in a read-only transaction (`db.View` + `txn.Get`).
- **Remove:** deletes the key with `txn.Delete`.
- **List:** walks the whole store with a Badger iterator (`it.Rewind / it.Valid / it.Next`, with `PrefetchSize = 10`), decodes each value into a `map[string]Entry`, and returns the keys sorted with `sort.Strings` so the output is alphabetical.

### Go concepts covered

This small project covers a lot of Go basics:

- Packages and imports, including a local sub-package (`training.go/dictionary/dictionary`)
- Structs, methods with pointer receivers, and the `Stringer` interface
- Error handling through multiple return values (`(Entry, error)`, `([]string, map[string]Entry, error)`)
- Closures passed as transaction callbacks (`func(txn *badger.Txn) error { ... }`)
- `defer` for cleaning up resources
- Maps, slices and sorting
- Binary encoding with `encoding/gob`
- Using a third-party module (BadgerDB) through Go modules

## Known limitations

This is learning code, so it has a few rough edges:

- Lookups are case-sensitive: `define Golang` will not find a word added as `golang`.
- `add`, `define` and `remove` do not check that you passed enough arguments, so leaving one out makes the program panic.
- An unknown `-action` value does nothing and prints no error.
- `strings.Title` is deprecated in recent Go versions. `golang.org/x/text/cases` is the modern replacement.
- The original course code had no `go.mod`, since it was written for GOPATH. One was added (module `training.go/dictionary`) so it builds with current Go toolchains.

## Credits

- Course: [Le langage Go : formation complète](https://www.udemy.com/course/le-langage-go-formation-complete/learn/) on Udemy
- Storage engine: [dgraph-io/badger](https://github.com/dgraph-io/badger)
