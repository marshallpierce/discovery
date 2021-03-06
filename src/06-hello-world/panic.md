# `panic!`

The `panic!` macro also sends its output to the ITM!

Change the `main` function to look like this:

``` rust
fn main() {
    panic!("Hello, world!");
}
```

Let's try this program. But before that let's update `.gdbinit` to run that `monitor` stuff for us
during GDB startup:

``` diff
 target remote :3333
 set print asm-demangle on
 load
+monitor tpiu config internal itm.txt uart off 8000000
+monitor itm port 0 on
 break hello_world::main
 continue
```

OK, now run it.

``` console
$ xargo run
(..)
Breakpoint 1, hello_world::main () at src/main.rs:8
8           panic!("Hello, world!");

(gdb) next
Program received signal SIGTRAP, Trace/breakpoint trap.
rust_begin_unwind (args=..., file=..., line=8, col=5) at aux/src/lib.rs:34
34          cortex_m::asm::bkpt();

(gdb) _
```

You'll see some new output in the `itmdump` terminal.

``` console
$ # itmdump terminal
(..)
PANIC at 'Hello, world!', src/main.rs:8:5
```

You won't get a `RUST_BACKTRACE` style backtrace in `itmdump`'s output, *but*
you can get the equivalent inside GDB. You already know the command:

```
(gdb) backtrace
#0  rust_begin_unwind (args=..., file=..., line=8, col=5) at aux/src/lib.rs:34
#1  0x08002548 in core::panicking::panic_fmt (fmt=..., file_line_col=<optimized out>) at $RUST_SRC/libcore/panicking.rs:71
#2  0x08002494 in core::panicking::panic (expr_file_line_col=<optimized out>) at $RUST_SRC/libcore/panicking.rs:51
#3  0x080001f6 in hello_world::main () at src/main.rs:8
```

Ultimately, `panic!` is just another function call so you can see it leaves behind a trace of
function calls.

Other interesting thing happened when we hit the `panic!` but you may have missed it. Let's re-run
the program but this time let's use `continue` instead of `next`:

```
(gdb) monitor reset halt
(..)
target halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x08000188 msp: 0x10002000

(gdb) continue
Continuing.

Breakpoint 1, hello_world::main () at src/main.rs:8
8           panic!("Hello, world!");
```

We are back in `main`, let's `continue`:

```
(gdb) continue
Continuing.

Program received signal SIGTRAP, Trace/breakpoint trap.
rust_begin_unwind (args=..., file=..., line=8, col=5) at aux/src/lib.rs:34
34          cortex_m::asm::bkpt();
```

> Program received signal SIGTRAP, Trace/breakpoint trap.

The program hit a breakpoint! But we didn't set one in GDB. What happened here is that `panic!`
called the `asm::bkpt()` function and that function executed the `BKPT` (breakpoint) instruction.

Remember that our microcontroller only supports 6 breakpoints? Well, `asm::bkpt()` *doesn't* count
towards that limit of 6. Only breakpoints set in GDB using the `break` command count towards the
limit. So, feel free to use the `auxN::bkpt()` function in your programs from now on.

Later on, we'll look into other simpler communication protocols.
