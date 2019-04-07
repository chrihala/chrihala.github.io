---
layout: post
title: My Radare2 Cheat Sheet
date: '2018-08-17T06:12:00.003-07:00'
author: Hland
tags:
- radare2
- reversing
- cheatsheet
modified_time: '2018-08-17T06:12:17.270-07:00'
blogger_id: tag:blogger.com,1999:blog-3264165303439231899.post-5263207092402720992
blogger_orig_url: https://blankhat.blogspot.com/2018/05/my-radare2-cheat-sheet.html
---

## Renaming

```
# rename a function
s <function_flag>
afvn [new_name]

# List local function variables
afv
# Rename variable
afvn [old_name] [new_name]

```

## Telescoping stack view
```
pxr @ esp
```

## Find symbols in libc

`dmi libc system`

## Search for strings 
`/ /bin/sh @ <address>`


## Some GDB Debugging tricks as well

Inspect the stack
```
x/100b $esp
x/100s $esp
x/100x $esp
```

Disassemble some code:
```
disassemble main
```

Set a breakpoint:
```
break *0xcafebabe
```

Go to debug mode:
```
ctrl^x ctrl^a
layout asm 
layout regs
```

## An out of place Strace Command

To trace only file access:
`strace -e trace=file ./utumno1.out testtest`

Similarly for network, process, ipc or memory:
`strace -e trace=network ./utumno1.out testtest`

`strace -e trace=process ./utumno1.out testtest`

`strace -e trace=ipc ./utumno1.out testtest`

`strace -e trace=memory ./utumno1.out testtest`

This is very useful for reversing what a binary is doing when there's no symbols and the debugger is failing us. 