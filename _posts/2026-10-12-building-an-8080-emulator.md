---
layout: post
title: "Building an Intel 8080 Emulator in C++"
date: 2026-10-12
categories: [systems, emulation]
tags: [cpp, intel-8080, computer-architecture]
---

I've been developing an Intel 8080 emulator in C++.

The project started as an opportunity to better understand
how processors execute instructions and manage their internal state.

## Architecture

The emulator separates the CPU and memory into distinct components.

This makes it easier to test instruction behavior independently.

## Instruction Decoding

One of the more interesting parts of the project has been
implementing the instruction decoder.

For example:

```cpp
void CPU::step() {
    uint8_t opcode = memory.read(pc++);
    execute(opcode);
}
