---
layout: post
title: "Writing an AVR Delay Generator in Rust"
date: 2026-11-01
categories: [systems]
tags: []
---

I came across a fairly simple problem while writing the firmware for my Intel 8080 power supervisor:

> I needed a delay.

This was the unfortunate moment when I realized that AVR assembly doesn't provide a convenient, built-in instruction for an arbitrary busy-wait delay.

So, I got out a pen and paper and scribbled down a few formulas. After consulting the AVR documentation, I came up with a set of general-purpose formulas for calculating the loop iterations needed to achieve a desired delay. Here are the formulas I started with.

### Converting a delay into clock cycles

Given a CPU clock frequency \(f\) in hertz and a requested delay \(t\) in nanoseconds, the ideal number of cycles is:

\[
C_{\text{ideal}} = \frac{f \times t}{1{,}000{,}000{,}000}
\]

Because instructions consume whole clock cycles, the generator rounds this value according to the selected rounding mode. The result is the target cycle count, \(C_{\text{target}}\).

### 8-bit delay loop

An 8-bit counter can execute between 1 and 256 iterations. With a counter initialized using `LDI`, then decremented using `DEC` and tested using `BRNE`, the loop consumes:

\[
C_{8} = 1 + n(1) + (n-1)(2) + 1 = 3n
\]

Here, \(n\) is the iteration count. The final `BRNE` takes one cycle rather than two because the branch is not taken.

When the generated routine saves and restores the counter register and status register (SREG), it uses another 10 cycles:

\[
C_{8,\text{inline}} = 3n + 10 + P
\]

\[
C_{8,\text{subroutine}} = 3n + 14 + P
\]

Here, \(P\) is the number of padding `NOP` instructions, each consuming one cycle. The subroutine formula adds four cycles for `RET` but does **not** include the caller's `CALL` or `RCALL` instruction.

### 16-bit delay loop

A 16-bit counter supports between 1 and 65,536 iterations. The counter is initialized using two `LDI` instructions, decremented using `SBIW`, and tested using `BRNE`:

\[
C_{16} = 2 + 2n + 2(n-1) + 1 = 4n + 1
\]

Preserving and restoring both counter registers and SREG adds 14 cycles:

\[
C_{16,\text{inline}} = 4n + 15 + P
\]

\[
C_{16,\text{subroutine}} = 4n + 19 + P
\]

For both counter widths, initializing the counter to zero represents the maximum iteration count: decrementing zero wraps the counter around before it eventually reaches zero again.

### Calculating the iteration count

Once the fixed overhead is accounted for, the generator chooses the largest whole number of iterations that does not exceed the requested cycle count:

\[
n_8 = \left\lfloor \frac{C_{\text{target}} - H_8}{3} \right\rfloor
\]

\[
n_{16} = \left\lfloor \frac{C_{\text{target}} - H_{16} - 1}{4} \right\rfloor
\]

For inline generation, \(H_8 = 10\) and \(H_{16} = 14\). For subroutine generation, add four cycles to each overhead to account for `RET`. These equations apply only when the target is large enough for the selected strategy and the calculated iteration count is within that counter's supported range.

Any cycles left over are filled with `NOP` instructions:

\[
P = C_{\text{target}} - C_{\text{generated without padding}}
\]

For very short delays, the generator can skip the counter loops entirely and emit only `NOP` instructions (and `RET` in subroutine mode).

These formulas assume classic AVR instruction timings and uninterrupted execution. Interrupts can extend the elapsed delay, and the subroutine timings exclude the cost of calling the routine.
