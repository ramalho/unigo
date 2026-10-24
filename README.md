# unigo

`unigo` is a `uniq` alternative that does not require the input to be sorted.

`unigo` reads lines from a file or standard input, and writes each distinct line
exactly once, in order of first appearance:

```sh
go run . testdata/input.txt

cat testdata/input.txt | go run .
```

## Why not `uniq`?

The standard `uniq` only detects adjacent duplicate lines, so the input has to
be sorted first.
`unigo` tracks every line it has seen,
so it works on unsorted input and preserves the original order.

The trade-off is memory:
`unigo` must store each unique line in main
memory, while `uniq` only needs to keep the
previous line.

> Note: `unigo` uses `bufio.Scanner` which caps line length at 64 KiB by default.
> See [How to read a text file line-by-line in Go when some lines are long enough to cause "bufio.Scanner: token too long" errors?](https://stackoverflow.com/questions/21124327/how-to-read-a-text-file-line-by-line-in-go-when-some-lines-are-long-enough-to-ca)


## How the deduplication logic works

The `WriteUnique` function does the real work. 
Lines previously seen are stored in a map
created like this:

```go
seen := make(map[string]struct{})
```

It is common practice in Go to declare a map of empty structs
to store values only in the keys, therefore using the map
as a primitive set, to ensure that each value is unique.

The heart of `WriteUnique` is a `for` loop over the input,
reading each line into the `line` variable.
For each line, the loop does 3 steps:

```go
before := len(seen)      #1: store length of `seen` 
seen[line] = struct{}{}  #2: put `line` into `seen`
if len(seen) > before {  #3: conditionally...
	fmt.Fprintln(buf, line)  # outuput line
}
```

In Go, `len` is not a function; it works more like a C macro:
the compiler generates inline the necessary code to get the
length of the value; no call stack to deal with. 
Therefore, applying `len` is very cheap.

The logic above does two `len` and one key lookup into the map.

In contrast, the usual way to use a map would be this:

```go
if _, found := seen[line]; !found {
	seen[line] = struct{}{}
	fmt.Fprintln(buf, line)
}
```

Here there are two lookups when the line is unique (never seen before).
Primitive benchmarks have shown that the solution with `len` is
slightly faster when most lines are unique and duplicates are rare.


## Building and testing

```sh
go build .
go test ./...
```

`main` is a thin wrapper over `run(cmdName, args, stdin, stdout, stderr) int`,
so the command is testable in-process without spawning a subprocess.
