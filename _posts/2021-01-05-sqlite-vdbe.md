---
layout: post
title: How to debug the Virtual Database Engine of SQLite
tags: [sqlite]
---

## What is the Virtual Database Engine of SQLite

Check [SQLite VDBE](https://sqlite.org/vdbe.html)

In short, it is an engine to execute the bytecode generated by the SQL processor in a virtual machine. The VDBE is the heart of SQLite, as occurs in the middle of the processing stream of SQLite:

![SQLite architecture](/assets/btree-arc.jpg)

## How to debug

The main implementation of VDBE is placed in `src/vdbe.c`. Most of a VDBE problem can be run in `int sqlite3VdbeExec(Vdbe *p)`.

In order to debug a VDBE problem, you can enable `-DSQLITE_DEBUG` when compiling, or you can print all the operations (including its opcode and parameters). It is very easy to do that, as the `int sqlite3VdbeExec(Vdbe *p)` is clear and easy to understand.

### Where are the Operation codes of VDBE

Documents about Opcode can be found here: https://www.sqlite.org/opcode.html

After compiling SQLite, in the build folder, there will be two files: `opcodes.h` and `opcodes.c`. All the operation names and their codes will be placed in both files.

### Important source files for debugging

- `src/vdbe.c`: contains the execution engine
- `build/opcodes.c`: contains all the operation names and codes