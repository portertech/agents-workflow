---
name: effective-go
description: "Apply Go best practices, idioms, and conventions to write clean, idiomatic Go code."
---

# Effective Go

Condensed reference from the official Effective Go guide. Apply these conventions when writing, reviewing, or refactoring Go code.

---

## Formatting

**Always use `gofmt`** - this is non-negotiable. The tool handles indentation, alignment, and spacing.

- **Indentation**: Tabs, not spaces
- **Line length**: No limit, but wrap long lines with extra tab indent
- **Parentheses**: Fewer than C/Java - control structures (`if`, `for`, `switch`) have no parentheses

```go
// gofmt aligns struct fields automatically
type T struct {
    name    string // name of the object
    value   int    // its value
}
```

---

## Naming

### Package Names
- **Short, concise, lowercase, single-word** - no underscores or mixedCaps
- Package name is base of source directory: `encoding/base64` → package `base64`
- Avoid repetition: `bufio.Reader` not `bufio.BufReader`

### Exported Names
- **MixedCaps** for exported, **mixedCaps** for unexported
- First letter determines visibility

### Getters/Setters
- Getter: `Owner()` not `GetOwner()`
- Setter: `SetOwner()`

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

### Interface Names
- One-method interfaces: method name + `-er` suffix
- `Reader`, `Writer`, `Formatter`, `Stringer`

### Method Names
- Honor canonical names: `Read`, `Write`, `Close`, `String` have specific signatures
- Your string-converter should be `String()` not `ToString()`

---

## Semicolons

Lexer inserts semicolons automatically. **Opening brace must be on same line**:

```go
// CORRECT
if i < f() {
    g()
}

// WRONG - semicolon inserted before brace
if i < f()  
{           
    g()
}
```

---

## Control Structures

### If

Use initialization statement to scope variables:

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

**Omit else** when body ends in `break`, `continue`, `goto`, or `return`:

```go
f, err := os.Open(name)
if err != nil {
    return err
}
// continue with f - no else needed
```

### For

Three forms (unifies `for` and `while`):

```go
for init; condition; post { }  // C-style
for condition { }              // while
for { }                        // infinite
```

**Range** for arrays, slices, strings, maps, channels:

```go
for key, value := range oldMap {
    newMap[key] = value
}

for key := range m { }           // key only
for _, value := range array { }  // value only (blank identifier)
```

**Parallel assignment** for multiple loop variables:

```go
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

### Switch

More flexible than C - no automatic fallthrough, cases can be comma-separated:

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

**Expression-less switch** (cleaner than if-else chains):

```go
switch {
case '0' <= c && c <= '9':
    return c - '0'
case 'a' <= c && c <= 'f':
    return c - 'a' + 10
}
```

**Type switch** for dynamic type discovery:

```go
switch t := value.(type) {
case bool:
    fmt.Printf("boolean %t\n", t)
case int:
    fmt.Printf("integer %d\n", t)
default:
    fmt.Printf("unexpected type %T\n", t)
}
```

---

## Functions

### Multiple Return Values

Use for value + error pattern:

```go
func (file *File) Write(b []byte) (n int, err error)
```

### Named Return Parameters

Document what's returned; enable bare `return`:

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return  // returns n, err
}
```

### Defer

Schedules function call for when enclosing function returns. **LIFO order**.

```go
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // runs on any return path
    // ... read file ...
}
```

Arguments evaluated when defer executes, not when deferred function runs.

---

## Data

### new vs make

| Function | Returns | Use For | Initializes |
|----------|---------|---------|-------------|
| `new(T)` | `*T` | Any type | Zeros memory |
| `make(T, args)` | `T` | Slices, maps, channels only | Initializes internal structure |

```go
p := new(SyncedBuffer)    // *SyncedBuffer, zeroed, ready to use
v := make([]int, 100)     // []int with len=100, initialized
m := make(map[string]int) // initialized map
c := make(chan int, 10)   // buffered channel
```

### Composite Literals

```go
return &File{fd: fd, name: name}  // labeled fields, any order
return &File{fd, name, nil, 0}    // positional, all fields required
```

**Zero value**: `new(File)` and `&File{}` are equivalent.

### Arrays vs Slices

**Arrays**: Value type, size is part of type, copying duplicates data.

**Slices**: Reference type, use for almost everything.

```go
func (f *File) Read(buf []byte) (n int, err error)  // slice parameter

n, err := f.Read(buf[0:32])  // read into first 32 bytes
```

**Append**:

```go
x := []int{1, 2, 3}
x = append(x, 4, 5, 6)      // must reassign - underlying array may change
x = append(x, y...)         // append another slice
```

### Maps

```go
m := make(map[string]int)
m["key"] = 42

value := m["key"]           // zero value if missing
value, ok := m["key"]       // comma-ok idiom
delete(m, "key")            // safe even if key missing
```

---

## Methods

### Pointer vs Value Receivers

- **Value receiver**: Method gets copy, can't modify receiver
- **Pointer receiver**: Method can modify receiver, more efficient for large structs

```go
type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte { ... }   // returns new slice
func (p *ByteSlice) Append(data []byte) { *p = ... }        // modifies in place
```

**Rule**: Value methods work on pointers and values; pointer methods only on pointers (but compiler auto-inserts `&` for addressable values).

---

## Interfaces

Interfaces specify behavior. **Small interfaces** (1-3 methods) are idiomatic.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

**Accept interfaces, return concrete types**:

```go
func NewReader(r io.Reader) *Reader  // accepts interface, returns concrete
```

### Type Assertions

```go
str := value.(string)              // panics if not string
str, ok := value.(string)          // safe - ok is false if fails
```

### Interface Checks at Compile Time

```go
var _ json.Marshaler = (*RawMessage)(nil)  // compile-time check
```

---

## Embedding

Compose types by embedding (not inheritance):

```go
type ReadWriter struct {
    *Reader  // embedded - methods promoted
    *Writer
}
```

Embedded type's methods become outer type's methods, but receiver is inner type.

```go
type Job struct {
    Command string
    *log.Logger  // can call job.Println()
}
```

---

## Concurrency

### Goroutines

Lightweight concurrent functions:

```go
go list.Sort()  // run concurrently

go func() {
    // function literal
}()
```

### Channels

```go
ch := make(chan int)        // unbuffered - synchronous
ch := make(chan int, 100)   // buffered

ch <- value   // send
value := <-ch // receive (blocks until data)
```

**Share by communicating**: Don't communicate by sharing memory; share memory by communicating.

### Common Patterns

**Signal completion**:

```go
done := make(chan bool)
go func() {
    // work...
    done <- true
}()
<-done  // wait for completion
```

**Semaphore** (limit concurrency):

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // acquire
    process(r)
    <-sem       // release
}
```

**Worker pool**:

```go
func worker(jobs <-chan Job, results chan<- Result) {
    for j := range jobs {
        results <- process(j)
    }
}

// Start N workers
for i := 0; i < N; i++ {
    go worker(jobs, results)
}
```

### Select

Multiplex channel operations:

```go
select {
case v := <-ch1:
    // received from ch1
case ch2 <- x:
    // sent to ch2
default:
    // no channel ready - non-blocking
}
```

---

## Errors

### Error Type

```go
type error interface {
    Error() string
}
```

### Return Errors, Don't Panic

```go
func Open(name string) (*File, error) {
    // ...
    if err != nil {
        return nil, &PathError{Op: "open", Path: name, Err: err}
    }
    return f, nil
}
```

### Error Strings

- Identify origin: prefix with package/operation
- Example: `"image: unknown format"`

### Checking Error Types

```go
if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
    // handle specific error
}
```

### Panic and Recover

**Panic**: For truly unrecoverable errors only. Avoid in libraries.

```go
panic("unreachable")
```

**Recover**: Only useful in deferred functions.

```go
func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

**Rule**: Convert panics to errors at package boundaries.

---

## Initialization

### Constants with iota

```go
type ByteSize float64

const (
    _  = iota             // ignore first
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
)
```

### init Functions

Called after all variable declarations, before `main`:

```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
}
```

---

## The Blank Identifier

Discard unwanted values:

```go
_, err := os.Stat(path)  // ignore first return value
```

**Import for side effect**:

```go
import _ "net/http/pprof"  // register handlers, don't use package
```

---

## Printing

| Format | Purpose |
|--------|---------|
| `%v` | Default format |
| `%+v` | Struct with field names |
| `%#v` | Go syntax representation |
| `%T` | Type of value |
| `%q` | Quoted string |

```go
fmt.Printf("%v\n", myStruct)    // default
fmt.Printf("%+v\n", myStruct)   // with field names
fmt.Printf("%#v\n", myStruct)   // Go syntax
```

### Stringer Interface

```go
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
```

**Avoid infinite recursion** - don't call `Sprintf` with `%s` on the receiver.

---

## Quick Reference

| Do | Don't |
|----|-------|
| `gofmt` all code | Manual formatting |
| `MixedCaps` / `mixedCaps` | `under_scores` |
| `Owner()` getter | `GetOwner()` |
| Small interfaces (1-3 methods) | Large interfaces |
| Return `error`, not panic | Panic in libraries |
| `make` for slices/maps/channels | `new` for slices/maps/channels |
| Share by communicating | Communicate by sharing |
| Named returns for documentation | Bare returns without named params |
| Check all errors | `_, _ = func()` |

---

*Condensed from [Effective Go](https://go.dev/doc/effective_go) - Last updated: February 2026*
