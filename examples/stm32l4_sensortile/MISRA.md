# MISRA C in the STM32L4 port

MISRA C is a set of rules for writing C code that behaves the same way every
time and on every compiler. The rules come from the car industry, and most
safety-related embedded code uses them today. Most rules forbid the parts of C
where the C standard leaves the result undefined or up to the compiler, or
where code that looks right does something else, e.g. an `if` without braces
or a pointer that reads an object as the wrong type.

The port's own code follows MISRA C:2012. The TinyMLDelta core in `runtime/`
is not part of the check.

The text below uses three terms:

| Term | Meaning |
|---|---|
| Rule | a requirement that a tool can check in the source code, e.g. Rule 15.6 says that every `if` body has braces |
| Deviation | a written exception to a rule, with a reason. MISRA expects real firmware to have a few, and it forbids exceptions that nobody wrote down. |
| Required or advisory | how strict a rule is. You need a deviation to break a required rule, and you should still write down why you break an advisory rule. |

## What the check covers

| Code | Checked | Reason |
|---|---|---|
| `src/main.c`, `src/board.c`, `src/tmd_port_stm32l4.c` | yes | the port's own code |
| `inc/*.h` | yes | as the files above include them |
| `runtime/` | no | the TinyMLDelta core, which every port shares |
| ST HAL, CMSIS, startup file | no | used exactly as ST ships them |

## How to run the check

```bash
make misra STM32CUBE=/path/to/STM32Cube_FW_L4
```

The command runs the MISRA addon of cppcheck 2.16 or newer. Each finding must
either be fixed in the code, or match a line in `misra/suppressions.txt` that
points to a deviation below, and any other finding makes the command fail.

cppcheck reports MISRA findings as "style" messages, so the Makefile passes
`--enable=style`. Without the flag, the addon prints nothing, and the check
looks like a pass. To test the check, we added a function with an `if` without
braces to a copy of `main.c`, and `make misra` failed as expected.

## Result

The first run reported 225 findings, and we fixed 137 of them in the code:

- Each function now returns from one place (Rule 15.5), and every `if` and
  loop has braces (Rule 15.6).
- Conditions are real `bool` values (Rule 14.4), and the code casts return
  values it ignores to `void` (Rule 17.7).
- Small helper functions read and write bytes in little-endian order, instead
  of `memcpy` between objects of different types (Rule 21.15). We also
  removed `memmove`, `memcmp` and `strlen`, because nothing used them.

The other 88 findings fall under the seven deviations below, and 52 of them
are unused macros in ST's HAL configuration template.

The changes did not change what the firmware does. We flashed the build after
the changes, and all 11 hardware tests in the README passed again.

## Deviations

### D-01, reading flash through a pointer (Rule 11.4)

The STM32 maps flash into the address space, so the code reads flash by
turning an address into a pointer. Only one function does the conversion,
`tmd_port_flash_ptr()`, and `main.c` calls the same function.

### D-02, void pointers in the port functions (Rule 11.5)

The `tmd_ports_t` table in the core fixes the signatures of
`flash_read(addr, void *dst, len)`, `flash_write(addr, const void *src, len)`
and `crc32(const void *buf, len)`. Each of the three functions converts its
`void *` argument to a byte pointer once, on its first line.

### D-03, ST register macros (Rule 11.4)

ST's macros, e.g. `__HAL_RCC_GPIOG_CLK_ENABLE()`, `GPIOG` and
`__HAL_FLASH_CLEAR_FLAG()`, cast fixed register addresses to pointers. CMSIS
and the HAL use the casts to reach the hardware registers.

### D-04, our own memcpy and memset (Rules 21.2 and 11.5)

The build links no C library, so `board.c` defines `memcpy` and `memset`. The
HAL, the core and the compiler call them by the standard names, and the C standard
fixes the names and the `void *` signatures.

### D-05, the printf-style logger (Rules 17.1 and 11.6)

The core's log function has the type `void (*log)(const char *fmt, ...)`, so
`board_log()` takes a variable number of arguments and uses `<stdarg.h>`. The
core prints pointers with `%p`, so the logger converts one pointer to a number
to print it. The logger only supports the format letters that the core uses.

### D-06, functions called from outside the check (Rule 8.7)

The hardware calls the exception handlers through the vector table in ST's
startup file, the startup file calls `__libc_init_array`, and the core calls
`tmd_layout()`. The check can't see any of the callers, so the tool reports
each function as used in only one file.

### D-07, ST's HAL configuration template (Rule 2.5)

`inc/stm32l4xx_hal_conf.h` is ST's template with a few settings changed. It
defines many macros that only ST's own HAL code reads.

## Limits of the check

- cppcheck's MISRA addon is not a certified checker, and a person has to
  review some of the rules. A safety project would add a qualified tool and a
  code review.
- MISRA sells the official rule texts, so the tool prints only the rule
  numbers.
- A clean MISRA check doesn't show that the code has no bugs. The hardware
  tests in the README check what the code does.
