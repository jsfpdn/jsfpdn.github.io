---
title: "Debugging Go Applications With Delve"
date: 2020-07-16T19:14:15+02:00
tags: ["programming", "go", "tooling"]
summary: " "
---

Even though I'm an avid user of Visual Studio Code,
it too has some pitfalls.
Recently, I've encountered some tomfoolery when debugging Go binaries
in VSCode, more specificaly,
some of the data structures that were being generated were not visible
in the debug window among other variables.

Eventhough the inspected struct was not that big,
the VSCode debugger failed to show the the contents of it's embedded map,
eventhough the map could be traced in the command line.
Clearly, the data was there, the debugger just hid it from me.
Was it because some compiler optimization I was not aware of?
Probably, but I could not be bothered...
[This brought my attention to **Delve**, a full featured command line debugger.](https://github.com/go-delve/delve)

## Delve

### Installation

Just `go get github.com/go-delve/delve/cmd/dlv`. Done.

### Basic Usage

Just run 
```text
dlv debug <target> [-- args]
````

with `<target>` being the relative path to the package you want to debug.
If you desire to debug tests, just replace `debug` with `test`.

Once I ran the command and subsequently ran `help`,
I was overwhelmed by all the options that were introduced to me.
Worry not, these are the most basic commands you'll need to know
to start debugging some code:

```text
(dlv) break <path/to/file>:<line_number> # set breakpoint in a file on a specific line
(dlv) continue # continue until a breakpoint is met or the program terminates
(dlv) locals   # print all of the variables in the local scope
(dlv) args     # print function arguments
(dlv) set      # change the value of a variable
(dlv) print <some_expression> # this will print a result of an expression
```

If you wish to debug some specific test in a specific file withouth other
tests being executed along the way, use 

```text
dlv test <target> -test.run ^<name_of_the_test>$
```

Some other interesting commands are

```text
(dlv) goroutines # list program goroutines
(dlv) thread     # print info for every thread
(dlv) regs       # print contents of CPU registers
(dlv) examine <addr> # examine memory of specified address
(dlv) stack      # print stack trace
```
