# C Language — Operating Systems & Memory Fundamentals Practice Questions

Use this as a learning checklist. Solve each question yourself, compile with warnings enabled (`gcc -Wall -Wextra -g`), test edge cases, and keep solutions in clearly named files. For exploitation exercises, use only isolated lab environments, CTF binaries, or programs you wrote yourself.

---

## 1. Language Fundamentals

### 1.1 Variables, Data Types, and Operators

- [ ] Declare variables of every primitive type (`char`, `short`, `int`, `long`, `long long`, `float`, `double`, `unsigned` variants) and print their sizes with `sizeof`.
- [ ] Print the minimum and maximum values of each integer type using `<limits.h>` constants (`INT_MIN`, `INT_MAX`, `UINT_MAX`, etc.).
- [ ] Explain the difference between `signed` and `unsigned` integers; demonstrate wrap-around behaviour with a code example.
- [ ] Write a program that uses all arithmetic operators (`+`, `-`, `*`, `/`, `%`) and explain integer truncation in division.
- [ ] Demonstrate all comparison operators (`==`, `!=`, `<`, `>`, `<=`, `>=`) and all logical operators (`&&`, `||`, `!`).
- [ ] Explain operator precedence with three non-obvious examples; use parentheses to force a different evaluation order.
- [ ] Use the ternary operator `? :` to write a compact `max(a, b)` expression.
- [ ] Write a program that demonstrates implicit type conversion (integer promotion, usual arithmetic conversions); then use explicit casts to control the result.
- [ ] Explain `static`, `extern`, `register`, `volatile`, and `const` storage-class and type qualifiers with one example each.
- [ ] Declare a `const` variable and a `volatile` variable; explain when each is critical for correct behaviour.

### 1.2 Control Flow

- [ ] Write `if`/`else if`/`else`, `switch`/`case`, `for`, `while`, and `do-while` examples for the same grade-calculator problem.
- [ ] Demonstrate `break`, `continue`, and `goto` with one example each; explain why `goto` is considered harmful and when it is acceptable.
- [ ] Write a nested loop that prints a multiplication table (1–10 × 1–10) aligned in columns using `printf` format specifiers.
- [ ] Use a `switch` statement on a `char` to build a simple calculator; handle the `default` case and fall-through deliberately with a comment.

### 1.3 Functions and Recursion

- [ ] Write functions for `is_even`, `factorial`, `fibonacci`, and `gcd` with proper prototypes in a header file.
- [ ] Explain the difference between passing by value and passing a pointer (pass by reference); demonstrate by writing `swap(int *a, int *b)`.
- [ ] Implement recursive and iterative versions of factorial and binary search; compare their call-stack depth.
- [ ] Write a recursive function that flattens a call tree (e.g., Tower of Hanoi); trace the stack frames by hand.
- [ ] Use `static` local variables inside a function to implement a counter that persists across calls; explain why this differs from a global variable.
- [ ] Declare a function pointer, assign a function to it, and call it; use an array of function pointers to implement a simple dispatch table.
- [ ] Write a variadic function using `<stdarg.h>` (`va_list`, `va_start`, `va_arg`, `va_end`) that sums an arbitrary number of integers.

### 1.4 Pointers and Pointer Arithmetic

- [ ] Declare a pointer, assign it the address of a variable with `&`, dereference it with `*`, and print both the address and the value.
- [ ] Explain the difference between `int *p`, `int* p`, `const int *p`, `int * const p`, and `const int * const p`.
- [ ] Write a program that increments a pointer through an integer array and prints each element without using array subscript notation.
- [ ] Demonstrate pointer arithmetic: adding an integer to a pointer, subtracting two pointers of the same type, and explain what the unit of arithmetic is.
- [ ] Write a function that takes `int *arr` and `int n` and reverses the array in-place using only pointer arithmetic (no indices).
- [ ] Explain dangling pointers and wild pointers with a code example for each; show how to avoid them.
- [ ] Explain `NULL` vs an uninitialized pointer vs a pointer to address `0`; write a safe null-check pattern.
- [ ] Write a double pointer (`int **pp`) example that dynamically allocates a 2D matrix and accesses elements.
- [ ] Use `void *` as a generic pointer: write a generic `swap` function that works for any type using `memcpy`.

### 1.5 Arrays and Strings

- [ ] Declare 1D and 2D integer arrays; initialize them at declaration and with loops; print them with nested loops.
- [ ] Explain why array names decay to pointers; show that `arr[i]` is identical to `*(arr + i)`.
- [ ] Write functions to find the minimum, maximum, sum, and average of an integer array using pointer parameters.
- [ ] Implement bubble sort, selection sort, and insertion sort on an integer array.
- [ ] Implement binary search on a sorted integer array (iterative and recursive).
- [ ] Declare a C string (`char []`), read it with `fgets`, and manipulate it with `strlen`, `strcpy`, `strncpy`, `strcat`, `strncat`, `strcmp`, `strncmp`, `strchr`, `strstr`, `strtok`.
- [ ] Explain why `gets` is dangerous and must never be used; show `fgets` as the safe replacement.
- [ ] Write `my_strlen`, `my_strcpy`, `my_strcat`, and `my_strcmp` from scratch without using `<string.h>`.
- [ ] Convert a string to an integer using `atoi`, `strtol`, and `sscanf`; explain why `strtol` is safer and how to detect errors.
- [ ] Reverse a C string in-place without using a second buffer.

### 1.6 Structs, Unions, Enums, and Typedefs

- [ ] Define a `struct Student` with name, age, and GPA; create instances, initialize them, and access members with `.` and `->`.
- [ ] Write a function that accepts a `struct` by value and another that accepts a pointer to the struct; explain the performance difference.
- [ ] Nest one struct inside another (e.g., `struct Address` inside `struct Person`) and access deeply nested members.
- [ ] Define a `union` that shares memory between an `int` and a `float`; print the raw bytes of each interpretation.
- [ ] Explain memory layout differences between `struct` and `union`; verify with `sizeof` and `offsetof`.
- [ ] Define an `enum` for days of the week and use it in a `switch` statement with a `default` case.
- [ ] Use `typedef` to alias a struct pointer type; explain why `typedef struct Node Node;` is common in linked-list code.
- [ ] Build a singly linked list using a self-referential struct; implement insert, delete, search, and print operations.
- [ ] Build a stack using a struct and a fixed-size array; implement `push`, `pop`, `peek`, and `is_empty`.
- [ ] Build a queue using a struct with head and tail pointers; implement `enqueue`, `dequeue`, and `is_empty`.

### 1.7 Bitwise Operations

- [ ] Apply all bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`) to integers and print the results in binary using a helper function.
- [ ] Use bitwise AND to check if a specific bit is set; use OR to set a bit; use AND with NOT to clear a bit; use XOR to toggle a bit.
- [ ] Implement a function that counts the number of set bits (Hamming weight / `popcount`) in an integer.
- [ ] Pack two 8-bit values into a 16-bit integer using shifts and masks; then unpack them back.
- [ ] Use bit fields inside a struct to pack multiple flags into a single byte; explain alignment and portability concerns.
- [ ] Implement multiplication and division by powers of 2 using left and right shifts; explain signed vs unsigned right shift behaviour.
- [ ] Use XOR to swap two integers without a temporary variable; explain when this trick is dangerous.

### 1.8 Macros, Preprocessing, and Header Files

- [ ] Define object-like macros for constants (`#define PI 3.14159`) and function-like macros (`#define MAX(a,b) ((a)>(b)?(a):(b))`); explain macro pitfalls (double evaluation, missing parentheses).
- [ ] Use `#ifdef`, `#ifndef`, `#if`, `#elif`, `#else`, `#endif` to conditionally compile debug logging code.
- [ ] Write a header guard (`#ifndef MY_HEADER_H ... #endif`) and explain why it prevents multiple-inclusion errors.
- [ ] Use `#pragma once` as an alternative to a header guard; explain portability trade-offs.
- [ ] Use the predefined macros `__FILE__`, `__LINE__`, `__func__`, `__DATE__`, and `__TIME__` in a debug-print macro.
- [ ] Explain the difference between `#include <header>` (system) and `#include "header"` (local); describe the search path for each.
- [ ] Use `#define` with the stringification operator `#` and the token-pasting operator `##`; show a practical use case.
- [ ] Write a multi-statement macro safely using the `do { ... } while(0)` idiom and explain why it is necessary.

### 1.9 Dynamic Memory Allocation

- [ ] Allocate an integer array on the heap with `malloc`; check the return value; use the array; free it; set the pointer to `NULL`.
- [ ] Allocate and zero-initialize memory with `calloc`; explain the difference from `malloc`.
- [ ] Resize an allocation with `realloc`; handle the case where `realloc` returns `NULL` without leaking the original block.
- [ ] Write a safe wrapper `xmalloc(size_t n)` that calls `malloc`, checks for `NULL`, and exits with an error message on failure.
- [ ] Allocate a 2D matrix on the heap as an array of row pointers; access elements; free all rows then the pointer array.
- [ ] Allocate a 2D matrix as a single contiguous block; explain how index arithmetic maps `[row][col]` to a flat offset.
- [ ] Demonstrate a memory leak using Valgrind; then fix the leak and confirm Valgrind reports zero leaks.
- [ ] Write a program with a use-after-free bug; detect it with AddressSanitizer (`-fsanitize=address`); then fix it.
- [ ] Write a program that double-frees a pointer; detect it with ASan; fix by setting the pointer to `NULL` after `free`.

### 1.10 Compilation, Linking, and Debugging

- [ ] Compile a single-file C program with `gcc -Wall -Wextra -Werror -g -o output main.c`; explain each flag.
- [ ] Split a program into `main.c`, `utils.c`, and `utils.h`; compile each `.c` to `.o` separately; link them into one executable.
- [ ] Write a `Makefile` with `all`, `clean`, and individual object-file targets; use automatic variables (`$@`, `$<`, `$^`).
- [ ] Use `gcc -E` to view preprocessed output; `gcc -S` to view assembly; `gcc -c` to produce an object file; then link manually.
- [ ] Use GDB to set a breakpoint, run the program, inspect variable values with `print`, step through with `next`/`step`, and examine memory with `x`.
- [ ] Use `gcc -fsanitize=address,undefined` to detect memory errors and undefined behaviour in a test program.
- [ ] Compile with different optimization levels (`-O0`, `-O1`, `-O2`, `-O3`, `-Os`) and compare the resulting binary sizes and execution times.

### 1.11 Standard I/O (`stdio.h`)

- [ ] Use `printf` with every format specifier: `%d`, `%i`, `%u`, `%o`, `%x`, `%X`, `%f`, `%e`, `%g`, `%c`, `%s`, `%p`, `%zu`, `%%`; explain the difference between `%d` and `%i`.
- [ ] Use width and precision modifiers: `%10d` (right-align), `%-10d` (left-align), `%010d` (zero-pad), `%.2f` (two decimal places), `%10.3f`.
- [ ] Read a single integer with `scanf`; read a string safely with `scanf("%49s", buf)` limiting to 49 characters; explain why unbounded `%s` is dangerous.
- [ ] Use `fgets` to read a full line (including spaces) from stdin; strip the trailing `\n` manually; compare with `gets` and explain why `gets` was removed in C11.
- [ ] Open a file with `fopen` in modes `"r"`, `"w"`, `"a"`, `"rb"`, `"wb"`, `"r+"`; explain each; always check the return value for `NULL`.
- [ ] Read a text file line by line with `fgets`; count lines, words, and characters; close with `fclose`.
- [ ] Write structured records to a binary file with `fwrite`; read them back with `fread`; verify data integrity by comparing the round-tripped values.
- [ ] Use `fseek`, `ftell`, and `rewind` to navigate inside an open file; read a specific record by index from a binary file.
- [ ] Use `fprintf` to write formatted output to a file; use `fscanf` to parse it back; explain why `fscanf` is fragile and when `fgets`+`sscanf` is safer.
- [ ] Use `sprintf` and `snprintf`; explain why `sprintf` is dangerous (buffer overflow) and `snprintf` is the safe replacement.
- [ ] Use `sscanf` to parse a formatted string (e.g., a log line `"2025-01-01 ERROR: code=42"`) into separate variables.
- [ ] Flush output explicitly with `fflush(stdout)`; explain when this is necessary (e.g., before `fork` or a `sleep`).
- [ ] Use `tmpfile()` to create a temporary file that is automatically deleted on close; explain the security advantage over `tmpnam`.
- [ ] Redirect stderr to a log file from within C by reopening `stderr` with `freopen`.

### 1.12 Error Handling and `errno`

- [ ] Explain `errno`: what it is, where it is defined (`<errno.h>`), when it is set, and why it must be checked immediately after a failing call.
- [ ] Use `perror("context")` to print a human-readable error message for the current `errno` value; compare with `fprintf(stderr, "%s\n", strerror(errno))`.
- [ ] Write a safe file-open wrapper that calls `fopen`, checks for `NULL`, prints `perror`, and exits with a non-zero status.
- [ ] Write a safe `malloc` wrapper (`xmalloc`) that checks for `NULL`, prints an error with `errno` details, and aborts.
- [ ] Handle `EINTR`: when a blocking syscall (`read`, `accept`, `waitpid`) returns `-1` with `errno == EINTR`, loop and retry; explain why `EINTR` happens.
- [ ] Distinguish between errors that can be retried (`EINTR`, `EAGAIN`, `EWOULDBLOCK`) and fatal errors (`EBADF`, `EFAULT`, `ENOMEM`).
- [ ] Use `assert` from `<assert.h>` to catch programming errors (precondition violations) during development; explain why assertions should not replace runtime error checks in production code.
- [ ] Write a multi-layer error propagation pattern: a low-level function returns `-1` on failure; a mid-level function checks and propagates it; the top level calls `perror` and exits.
- [ ] Use `atexit()` to register a cleanup function that runs on normal program exit; explain why it does not run after `_exit()` or a signal.

---

## 2. Standard Library Deep Dive

### 2.1 `<stdlib.h>`

- [ ] Use `atoi`, `atol`, `atof`; explain why they are unsafe (no error detection) and always prefer `strtol`, `strtoul`, `strtod` instead.
- [ ] Use `strtol(str, &endptr, base)`: parse base-10, base-16, and base-2 integers; check `endptr` and `errno` to detect invalid input.
- [ ] Use `malloc`, `calloc`, `realloc`, and `free`; explain the difference between `calloc` (zero-initializes) and `malloc` (uninitialized).
- [ ] Use `exit(EXIT_SUCCESS)` and `exit(EXIT_FAILURE)`; explain `atexit` handler ordering and why `_exit` skips them.
- [ ] Use `abs`, `labs`, `llabs`, and `div`/`ldiv` for integer arithmetic; explain why `abs` on a negative `INT_MIN` is undefined behaviour.
- [ ] Use `qsort` with a custom comparator to sort an array of integers, then an array of structs by a field.
- [ ] Use `bsearch` on a sorted array with the same comparator as `qsort`; explain what it returns when the key is not found.
- [ ] Use `rand` and `srand`; explain why `rand()` is not suitable for security or cryptography; demonstrate a reproducible sequence with a fixed seed.
- [ ] Use `getenv` to read environment variables; explain the security risk of trusting environment variables in setuid programs.
- [ ] Use `system(cmd)` and explain why it is dangerous in programs with elevated privileges; state when `fork`+`exec` is always preferred.

### 2.2 `<string.h>`

- [ ] Use `strlen`, `strcpy`, `strncpy`, `strcat`, `strncat`, `strcmp`, `strncmp`, `strchr`, `strrchr`, `strstr`, `strtok`, and `memset` in one program.
- [ ] Explain the `strncpy` trap: it does not guarantee NUL-termination when the source is longer than `n`; always manually NUL-terminate.
- [ ] Use `memcpy` and `memmove`; explain why `memcpy` has undefined behaviour on overlapping buffers and `memmove` does not.
- [ ] Use `memcmp` to compare binary buffers; explain why `strcmp` must not be used on data that may contain embedded NUL bytes.
- [ ] Implement `my_strlen`, `my_strcpy`, `my_strcat`, `my_strcmp`, and `my_memcpy` from scratch without using `<string.h>`.
- [ ] Use `strtok_r` (reentrant version of `strtok`) and explain why `strtok` is not thread-safe.
- [ ] Use `memset` to zero-out a sensitive buffer (e.g., a password) before `free`; explain why the compiler may optimize it away and how `explicit_bzero` or `memset_s` prevents that.

### 2.3 `<ctype.h>`

- [ ] Use `isalpha`, `isdigit`, `isalnum`, `isspace`, `isupper`, `islower`, `ispunct`, `isprint`, `iscntrl` on characters in a string; explain why the argument must be cast to `unsigned char`.
- [ ] Use `toupper` and `tolower` to convert a string in-place; explain the `unsigned char` cast requirement.
- [ ] Write a string tokenizer using `isspace` as the delimiter predicate.
- [ ] Write an input validator that accepts only alphanumeric characters and rejects everything else using `isalnum`.
- [ ] Explain why `ctype.h` functions are locale-sensitive and what that means for security-critical parsing.

### 2.4 `<math.h>`

- [ ] Use `sqrt`, `pow`, `fabs`, `ceil`, `floor`, `round`, `fmod`, `log`, `log2`, `log10`, `exp`, and `sin`/`cos`/`tan` in meaningful examples; compile with `-lm`.
- [ ] Explain `NaN` and `Inf` in floating-point; use `isnan`, `isinf`, and `isfinite` from `<math.h>` to detect them.
- [ ] Explain `DBL_EPSILON` from `<float.h>`; use it to write a safe floating-point equality comparison.
- [ ] Use `modf` to split a `double` into its integer and fractional parts.

### 2.5 `<time.h>`

- [ ] Use `time(NULL)` to get the current Unix timestamp; convert it to a human-readable string with `ctime` and `asctime`.
- [ ] Use `localtime` and `gmtime` to convert a `time_t` to a `struct tm`; format it with `strftime`.
- [ ] Use `mktime` to convert a `struct tm` back to a `time_t`; compute the number of days between two dates.
- [ ] Use `clock()` to benchmark a loop; convert the result to seconds with `CLOCKS_PER_SEC`.
- [ ] Use `clock_gettime(CLOCK_MONOTONIC, &ts)` for high-resolution timing that is immune to system clock changes; compare with `gettimeofday`.
- [ ] Use `nanosleep` to sleep for a precise duration; handle `EINTR` by checking the remaining time and retrying.

### 2.6 `<stdint.h>` and `<inttypes.h>`

- [ ] Use `int8_t`, `int16_t`, `int32_t`, `int64_t`, `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t` in a struct representing a network packet header.
- [ ] Explain the difference between fixed-width types (`int32_t`), minimum-width types (`int_least32_t`), and fastest types (`int_fast32_t`).
- [ ] Use `SIZE_MAX`, `UINT32_MAX`, `INT64_MIN`, `INT64_MAX` from `<stdint.h>` in overflow-guard checks.
- [ ] Use `PRId32`, `PRIu64`, `SCNd32` macros from `<inttypes.h>` to portably `printf` and `scanf` fixed-width integers.
- [ ] Use `uintptr_t` to safely cast a pointer to an integer and back; explain why casting to `int` or `long` is not portable.

### 2.7 `<assert.h>` and `<errno.h>`

- [ ] Use `assert(condition)` to document and enforce preconditions in functions; show that `NDEBUG` disables all assertions at compile time.
- [ ] Use `static_assert` (C11) to enforce compile-time constraints (e.g., `sizeof(struct Header) == 16`).
- [ ] Write a function that checks `errno` before and after a syscall; explain that `errno` is only meaningful after a call that returns an error indicator.
- [ ] List 10 important `errno` values (`ENOENT`, `EACCES`, `EBADF`, `EINVAL`, `ENOMEM`, `EEXIST`, `EBUSY`, `EINTR`, `EAGAIN`, `EPIPE`) and give a scenario for each.

---

## 3. Mini Projects (Fundamentals)

- [ ] Build a menu-driven calculator: accept two numbers and an operator from the user; handle division by zero; loop until the user chooses to exit.
- [ ] Build an ATM simulator: track balance in a struct; implement deposit, withdrawal (reject overdrafts), and balance display; store state to a binary file and reload on startup.
- [ ] Build a student grade manager: store name, marks, and attendance in an array of structs; compute grade (A–F), percentage, and pass/fail; sort by marks using `qsort`; save to CSV.
- [ ] Build a contact book: store contacts (name, phone, email) in a dynamically grown array; support add, search by name, delete, list all, and save/load to a text file.
- [ ] Build a command-line todo list: read tasks from a file on startup; support add, complete, delete, and list commands via `argv`; persist changes back to the file.
- [ ] Build a word frequency counter: read a text file; tokenize with `strtok`; count each word in a hash-table (open addressing or chaining); print the top 10 words.
- [ ] Build a number base converter: accept a number and source base (2–16) via `argv`; convert to all other bases and print; validate input with `strtol` and `errno`.
- [ ] Build a string utilities library (`strlib.c` / `strlib.h`): implement `str_trim`, `str_split`, `str_join`, `str_replace`, and `str_to_upper`; write a separate `test_strlib.c` that exercises every function.

---

## 4. Memory Management

### 4.1 Process Memory Layout

- [ ] Draw and explain the five segments of a Linux process: text (`.text`), initialized data (`.data`), BSS (`.bss`), heap, and stack.
- [ ] Write programs that place variables in each segment: a global initialized variable (`.data`), a global uninitialized variable (`.bss`), a local variable (stack), and a `malloc`-ed value (heap); verify with `/proc/<pid>/maps`.
- [ ] Explain why the stack grows downward and the heap grows upward on x86/x86-64 Linux.
- [ ] Use `size` (from binutils) on a compiled binary to read its `.text`, `.data`, and `.bss` segment sizes.
- [ ] Read `/proc/self/maps` from inside a running C program and print the memory regions with their permissions.

### 4.2 Stack and Stack Frames

- [ ] Explain what a stack frame contains: return address, saved frame pointer, local variables, and function arguments (for cdecl).
- [ ] Write a recursive function and use GDB's `backtrace` (or `bt`) command to view the chain of stack frames.
- [ ] Use GDB to inspect the stack pointer (`$rsp`/`$esp`) and frame pointer (`$rbp`/`$ebp`) at a breakpoint.
- [ ] Demonstrate that local variables are stored on the stack by printing their addresses in nested function calls; observe the decreasing addresses.
- [ ] Write a function with a large local array and observe stack usage with `ulimit -s`; trigger a stack overflow and explain the crash.

### 4.3 Heap Management

- [ ] Explain how `malloc`/`free` use the heap: chunk headers, free lists, coalescing of adjacent free blocks.
- [ ] Use Valgrind (`valgrind --leak-check=full`) on a program with a deliberate leak, invalid read, and invalid write; interpret each report line.
- [ ] Use AddressSanitizer to detect a heap buffer overflow, use-after-free, and double-free; show the `SUMMARY` line for each.
- [ ] Explain the difference between a memory leak (allocated but unreachable) and memory that is still reachable at exit.
- [ ] Write a simple arena allocator: allocate a large block once, hand out slices linearly, and reset the arena with a single pointer reset.

### 4.4 Memory Alignment and Padding

- [ ] Print the `sizeof` and `offsetof` each member of a struct; explain where padding bytes are inserted and why.
- [ ] Reorder struct members to minimize padding; verify the size reduction with `sizeof`.
- [ ] Use `__attribute__((packed))` to remove padding; explain the performance and portability consequences.
- [ ] Explain natural alignment rules: a `int` (4 bytes) must be at an address divisible by 4; verify with a `%p` print.
- [ ] Explain `alignas` and `alignof` (C11); use `posix_memalign` or `aligned_alloc` to allocate over-aligned memory for SIMD.

### 4.5 Calling Conventions

- [ ] Explain the System V AMD64 ABI: which registers hold the first six integer arguments (`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`), which hold the return value (`rax`), and which are callee-saved vs caller-saved.
- [ ] Write a simple function and disassemble it with `objdump -d`; identify the function prologue (push rbp; mov rbp,rsp) and epilogue (pop rbp; ret).
- [ ] Explain `cdecl`: caller cleans the stack, arguments pushed right to left; compare with `stdcall`: callee cleans the stack.
- [ ] Use GDB to inspect register values at a function call boundary; verify which registers contain the arguments.
- [ ] Explain what happens when a function returns a struct larger than two registers (returned via a hidden pointer argument).

---

## 5. System Programming

### 5.1 File I/O and File Descriptors

- [ ] Explain file descriptors: 0 (stdin), 1 (stdout), 2 (stderr); show that they are small integers returned by `open`.
- [ ] Open a file with `open`, read from it with `read`, write to it with `write`, and close it with `close`; check every return value.
- [ ] Explain the difference between the POSIX API (`open`/`read`/`write`) and the C standard library API (`fopen`/`fread`/`fwrite`); explain buffering.
- [ ] Use `lseek` to seek to a specific byte offset; use `SEEK_SET`, `SEEK_CUR`, and `SEEK_END`.
- [ ] Use `dup` and `dup2` to redirect stdout to a file within a process; restore it afterward.
- [ ] Use `fcntl` to get and set file descriptor flags (e.g., `O_NONBLOCK`); explain what non-blocking I/O means.
- [ ] Explain Linux file permissions (owner/group/other, read/write/execute); use `chmod`, `chown`, `stat`, and `access` from C.
- [ ] Use `stat`/`fstat` to retrieve file metadata (size, permissions, modification time) and print it in human-readable form.

### 5.2 Processes

- [ ] Use `fork()` to create a child process; print the PID of both parent and child; explain what is copied and what is shared.
- [ ] Use `wait()` and `waitpid()` to collect a child's exit status; explain zombie processes and how to prevent them.
- [ ] Use `exec()` family (`execl`, `execlp`, `execv`, `execvp`, `execve`) to replace the current process image; explain what is preserved across `exec`.
- [ ] Combine `fork` + `exec` to implement a mini shell that runs one command; collect the exit status with `waitpid`.
- [ ] Use `getpid()`, `getppid()`, `getuid()`, `geteuid()` and print the results; explain real vs effective user ID.
- [ ] Use `exit()`, `_exit()`, and `atexit()` handlers; explain when `_exit` is needed (inside a forked child that exec-fails).
- [ ] Use `system()` and explain why it is dangerous in privileged programs (it invokes a shell); prefer `fork`+`exec`.

### 5.3 Signals

- [ ] List the most important POSIX signals: `SIGINT`, `SIGTERM`, `SIGKILL`, `SIGSEGV`, `SIGFPE`, `SIGALRM`, `SIGCHLD`, `SIGUSR1`, `SIGUSR2`.
- [ ] Register a custom signal handler with `signal()` for `SIGINT`; gracefully shut down a loop when Ctrl+C is pressed.
- [ ] Use `sigaction()` instead of `signal()` and explain why `sigaction` is safer and more portable.
- [ ] Use `kill()` to send a signal from a parent process to a child; use `raise()` to send a signal to yourself.
- [ ] Use `sigprocmask` to block signals during a critical section; unblock them afterward; explain `SA_RESTART`.
- [ ] Use `alarm()` and `SIGALRM` to implement a timeout for a blocking operation.
- [ ] Explain async-signal-safe functions: list functions that are safe to call inside a signal handler and ones that are not.

### 5.4 Pipes and Named Pipes (FIFO)

- [ ] Create an anonymous pipe with `pipe()`; write a string in the parent and read it in the child after `fork`.
- [ ] Implement a `parent → child` data pipeline: redirect the write end of a pipe to the child's stdin using `dup2`.
- [ ] Chain two child processes with two pipes to replicate `cmd1 | cmd2` (like a shell pipeline).
- [ ] Create a named pipe (FIFO) with `mkfifo()`; open it in one program for writing and another for reading; demonstrate bidirectional IPC.
- [ ] Explain the difference between anonymous pipes (require `fork`) and named pipes (work between unrelated processes).
- [ ] Handle `SIGPIPE` (broken pipe) gracefully: write to a pipe whose read end has been closed.

### 5.5 POSIX Threads (pthreads)

- [ ] Create a thread with `pthread_create`; pass an argument; join it with `pthread_join`; print from both threads.
- [ ] Demonstrate a data race: two threads increment a shared counter without synchronization; show the wrong result.
- [ ] Fix the data race with a `pthread_mutex_t`; use `pthread_mutex_lock` and `pthread_mutex_unlock` correctly.
- [ ] Use `pthread_cond_t` (condition variables) with a mutex to implement a producer–consumer buffer.
- [ ] Use `pthread_rwlock_t` to allow multiple concurrent readers but exclusive writers on a shared data structure.
- [ ] Use thread-local storage (`__thread` or `pthread_key_create`) to give each thread its own private copy of a variable.
- [ ] Implement a thread pool: a fixed number of worker threads pull tasks from a shared queue; the main thread enqueues tasks and waits for results.
- [ ] Demonstrate a deadlock between two mutexes: thread A holds lock 1 and waits for lock 2; thread B holds lock 2 and waits for lock 1; explain and fix it.
- [ ] Use `pthread_once` to safely initialize a shared resource exactly once across multiple threads.

### 5.6 Shared Memory and IPC

- [ ] Create a POSIX shared memory object with `shm_open` + `ftruncate` + `mmap`; write from one process, read from another; clean up with `munmap` + `shm_unlink`.
- [ ] Use a POSIX semaphore (`sem_open` / `sem_wait` / `sem_post` / `sem_unlink`) to synchronize access to shared memory between two processes.
- [ ] Create a POSIX message queue with `mq_open`; send a message from one process and receive it in another; clean up with `mq_unlink`.
- [ ] Explain the difference between pipes, shared memory, message queues, and sockets as IPC mechanisms — state the trade-offs of each.
- [ ] Implement a producer–consumer example using POSIX shared memory + semaphores (no threads — use two separate processes).

### 5.7 TCP/UDP Sockets

- [ ] Write a TCP server that binds to `127.0.0.1`, listens, accepts one connection, echoes data, and closes; write a matching client.
- [ ] Extend the TCP server to handle multiple clients using `fork` (one child per connection); collect zombies with `SIGCHLD`.
- [ ] Extend the TCP server to handle multiple clients using `select` or `poll` without forking; explain the difference from the forked version.
- [ ] Write a UDP server and client on localhost; send and receive a datagram; explain why UDP is connectionless.
- [ ] Use `getaddrinfo` and `getnameinfo` for portable hostname/port resolution; explain why they replace `gethostbyname`.
- [ ] Set socket options with `setsockopt`: `SO_REUSEADDR`, `SO_KEEPALIVE`, and a receive timeout `SO_RCVTIMEO`; explain each.
- [ ] Use `send`/`recv` instead of `write`/`read` on a socket and explain the extra flags parameter.
- [ ] Implement a simple line-based protocol on top of TCP: frame messages with a newline delimiter; handle partial reads correctly.

### 5.8 Linux System Calls

- [ ] Use `strace` on a simple program (`strace ./hello`) and read every system call it makes; explain the most common ones.
- [ ] Explain the system call interface: user space → `syscall` instruction → kernel → return; identify the syscall number mechanism.
- [ ] Write a program that calls `write` directly via the `syscall()` function (bypassing libc) and explain the register ABI.
- [ ] Use `ltrace` to intercept library calls; compare with `strace` (kernel calls); explain when each is more useful.
- [ ] Use `mmap` to map a file into memory; read and modify it through the pointer; use `msync` to flush changes.
- [ ] Use `mprotect` to change memory region permissions (e.g., remove write permission from a page) and observe the effect.
- [ ] Explain `brk` and `sbrk` and how early `malloc` implementations used them before `mmap` became preferred.

---

## 6. Compiler and Binary Fundamentals

### 6.1 GCC and Clang

- [ ] Compile with `gcc -Wall -Wextra -Wpedantic -Wshadow -Wformat=2 -g`; fix every warning the compiler emits.
- [ ] Compile the same source with Clang (`clang -Wall -Wextra`) and compare the diagnostic messages with GCC's.
- [ ] Use `gcc -std=c99` and `gcc -std=c11`; explain the key additions of C99 (VLAs, `//` comments, `stdint.h`) and C11 (`_Alignas`, `_Atomic`, `_Generic`).
- [ ] Use `-D NAME=VALUE` on the command line to define a macro; use it to enable debug logging without modifying source.
- [ ] Use `-I path` to add an include search directory and `-L path` / `-l name` to link a library.
- [ ] Generate a dependency file with `-MMD -MP` and include it in a `Makefile` to enable automatic header-change rebuilds.

### 6.2 Makefiles

- [ ] Write a `Makefile` for a multi-file project with `CC`, `CFLAGS`, `LDFLAGS`, object-file rules, and `clean`.
- [ ] Use automatic variables: `$@` (target), `$<` (first prerequisite), `$^` (all prerequisites) inside Makefile rules.
- [ ] Use a pattern rule (`%.o: %.c`) to compile all `.c` files uniformly without repeating rules.
- [ ] Use `$(wildcard *.c)` and `$(patsubst %.c,%.o,...)` to automatically detect source files.
- [ ] Add a `run` target, a `test` target, and a `valgrind` target to a Makefile.
- [ ] Explain phony targets (`.PHONY: all clean`) and why they are necessary when a file named `clean` might exist.

### 6.3 Static vs Dynamic Linking

- [ ] Compile two programs — one linked statically (`-static`) and one dynamically — and compare sizes with `ls -lh`.
- [ ] Use `ldd` on a dynamically linked binary to list all shared library dependencies and their resolved paths.
- [ ] Create a static library (`.a`) with `ar rcs libmylib.a utils.o`; link it into a program with `-L. -lmylib`.
- [ ] Create a shared library (`.so`) with `gcc -fPIC -shared`; set `LD_LIBRARY_PATH` to find it at runtime.
- [ ] Explain Position Independent Code (`-fPIC`) and why it is required for shared libraries.
- [ ] Use `LD_PRELOAD` to inject a custom shared library at runtime (e.g., override `malloc`); explain the security implications.

### 6.4 ELF Binary Format

- [ ] Use `readelf -h` to inspect the ELF header of a compiled binary; identify the magic bytes, class (32/64-bit), architecture, and entry point.
- [ ] Use `readelf -S` to list all sections (`.text`, `.data`, `.bss`, `.rodata`, `.symtab`, `.strtab`, `.plt`, `.got`); describe the purpose of each.
- [ ] Use `readelf -l` to list program headers (segments); explain the difference between ELF sections and ELF segments.
- [ ] Use `readelf -d` to display the dynamic section of a shared-library-linked binary; identify `NEEDED`, `RPATH`, and `RUNPATH` entries.
- [ ] Use `nm` to list symbols in an object file and a binary; explain the letter codes (T = text, D = data, B = bss, U = undefined, u = unique global).
- [ ] Use `objdump -d` to disassemble the `.text` section of a binary; identify `main`, function calls (`call`), and the function epilogue (`ret`).
- [ ] Use `objdump -x` to display all headers; use `objdump -s -j .rodata` to dump the contents of `.rodata` as hex+ASCII.
- [ ] Use `strings` on a binary to extract all printable character sequences; explain what kinds of secrets and clues strings can reveal.
- [ ] Use `file` and `checksec` on a binary and interpret all reported mitigations (Canary, NX, PIE, RELRO, FORTIFY).

### 6.5 Debugging with GDB

- [ ] Start GDB (`gdb ./program`), set a breakpoint at `main` with `break main`, run with `run`, and inspect local variables with `info locals`.
- [ ] Step one source line with `next`, step into a function with `step`, continue to the next breakpoint with `continue`, and run until the current function returns with `finish`.
- [ ] Examine memory with `x/Nuf addr`: print `N` units of format `u` (e.g., `x/16xb $rsp` prints 16 hex bytes at the stack pointer).
- [ ] Use `info registers` to print all CPU registers; use `print $rip` to print the instruction pointer.
- [ ] Disassemble a function in GDB with `disassemble main`; set a breakpoint at a specific address with `break *0xADDR`.
- [ ] Use `watch` to set a watchpoint on a variable; GDB will stop whenever the variable's value changes.
- [ ] Use `backtrace` (`bt`) to view the call stack after a crash (segfault); use `frame N` to switch to a specific frame and inspect its variables.
- [ ] Load a core dump (`gdb ./program core`); use `bt` and `info registers` to determine the crash location.
- [ ] Use GDB Python scripting or GEF/pwndbg commands to pretty-print heap chunks (`heap chunks`) and check stack canaries.
- [ ] Set a conditional breakpoint (`break func if x == 0`) to stop only when a specific condition is true.

---

## 7. Memory Exploitation Fundamentals

> ⚠️ All exercises in this section must be performed **only** against programs you wrote, CTF binaries, intentionally vulnerable training targets, or lab environments you own and control. Never test against systems or binaries you do not have explicit written authorization to test.

### 7.1 Buffer Overflows

- [ ] Write a deliberately vulnerable program with `strcpy` into a fixed-size stack buffer; compile without mitigations (`-fno-stack-protector -z execstack -no-pie`); overflow the buffer and observe the crash in GDB.
- [ ] Use GDB to find the exact offset from the buffer start to the saved return address by examining the stack layout.
- [ ] Explain what happens to the instruction pointer (`$rip`/`$eip`) when the return address is overwritten with a controlled value.
- [ ] Distinguish stack buffer overflows from heap buffer overflows; explain which control-flow data each can corrupt.
- [ ] Write an intentionally vulnerable program that reads from stdin with `gets` or `scanf("%s", buf)`; demonstrate the overflow.
- [ ] Explain why modern compilers and OS features (canaries, ASLR, NX) make exploitation harder; verify each mitigation with `checksec`.

### 7.2 Integer Overflows

- [ ] Write a program where adding two large `unsigned int` values wraps around to zero; use this to bypass a size check.
- [ ] Demonstrate a signed integer overflow that changes the sign of a value used as an array index.
- [ ] Show that `size_t` overflow can cause `malloc` to allocate a tiny buffer when the intended size was very large.
- [ ] Use `-fsanitize=undefined` (UBSan) to detect signed integer overflow automatically; explain which overflows are defined vs undefined in C.
- [ ] Explain integer truncation: assigning a `long` to a `short` or `int`; demonstrate data loss.

### 7.3 Format String Vulnerabilities

- [ ] Write a program with `printf(user_input)` (no format string); pass `%s %s %s` as input and observe the crash or garbage output.
- [ ] Explain what `%n` does in `printf`; explain why user-controlled format strings are a write-anywhere primitive.
- [ ] Show the safe fix: always use `printf("%s", user_input)` and never pass untrusted data as the format string.
- [ ] Use `%08x` or `%p` repeated in a format string to leak stack values in a vulnerable test program.
- [ ] Use `-Wformat-security` and `-Wformat=2` compiler flags; explain the warnings they produce for unsafe `printf` calls.

### 7.4 Use-After-Free and Double Free

- [ ] Write a program that `free`s a pointer and then reads through it (use-after-free); detect with ASan.
- [ ] Write a program that calls `free` twice on the same pointer (double free); detect with ASan; fix by setting to `NULL` after `free`.
- [ ] Explain how UAF and double-free bugs are exploitable: the freed chunk can be reallocated for a different object, leading to type confusion.
- [ ] Use Valgrind to detect invalid reads/writes to freed memory in a test program.

### 7.5 Null Pointer Dereference

- [ ] Write a program that dereferences a `NULL` pointer; explain the resulting `SIGSEGV` signal.
- [ ] Show a common pattern: `malloc` returns `NULL` on failure; failing to check and then dereferencing it causes a crash.
- [ ] Explain how, on some older kernels, a null dereference near offset 0 of a mapped page could be exploited; explain why this is prevented by modern `vm.mmap_min_addr`.

### 7.6 Heap Corruption

- [ ] Write a program that writes one byte past the end of a heap-allocated buffer (off-by-one); detect with ASan.
- [ ] Explain how off-by-one errors on the heap can corrupt adjacent chunk metadata, leading to arbitrary writes.
- [ ] Explain heap chunk structure in ptmalloc2 (glibc): `prev_size`, `size`, flags (P, M, A), forward/backward pointers in free lists.
- [ ] Use GDB with GEF or pwndbg's `heap chunks` command to inspect live heap chunk metadata.

### 7.7 Race Conditions and TOCTOU

- [ ] Write a multithreaded program with a race condition on a global counter; verify the wrong result without synchronization; fix with a mutex.
- [ ] Explain a TOCTOU vulnerability: a program checks a file's existence with `access()`, then opens it with `open()` — an attacker can replace the file between the two calls.
- [ ] Write a C program that demonstrates TOCTOU on a temporary file; explain the safe fix using `open` with `O_CREAT | O_EXCL`.
- [ ] Use `strace` to observe the `access` → `open` sequence in a vulnerable program.

### 7.8 Mitigations

- [ ] **Stack Canaries:** Compile with `-fstack-protector-strong`; use GDB to find the canary value on the stack; explain what happens when it is overwritten.
- [ ] **NX / DEP:** Compile without `-z execstack`; attempt to place shellcode on the stack and explain why it fails with NX enabled; verify with `checksec`.
- [ ] **ASLR:** Enable/disable ASLR with `/proc/sys/kernel/randomize_va_space`; print `main`'s address across multiple runs to observe randomization.
- [ ] **PIE:** Compile with and without `-no-pie`; observe that PIE binaries have randomized base addresses each run.
- [ ] **RELRO:** Compile with `-Wl,-z,relro,-z,now` (Full RELRO); use `readelf -d` to verify; explain what Full RELRO prevents vs Partial RELRO.
- [ ] **Safe practices:** Use `strncpy`, `snprintf`, `fgets` instead of unsafe equivalents; add bounds checks before every array access; always check `malloc` return values; set pointers to `NULL` after `free`.

---

## 8. Essential Tools

### 8.1 GDB and Extensions

- [ ] Use vanilla GDB to debug a segfault: load a coredump, run `bt`, inspect registers, identify the crashing instruction.
- [ ] Install GEF or pwndbg; use `context` to see registers, stack, and disassembly simultaneously at each breakpoint.
- [ ] Use GEF/pwndbg's `heap chunks` to inspect live heap allocations; use `vmmap` to print memory regions.
- [ ] Use `pattern create` and `pattern search` (cyclic patterns) to find the exact offset to the return address in a buffer overflow exercise.
- [ ] Use GDB's TUI mode (`gdb -tui`) or `layout regs` / `layout asm` to view source/assembly alongside registers.

### 8.2 Valgrind

- [ ] Run `valgrind --leak-check=full --show-leak-kinds=all ./program` on a leaky program; interpret every line of output.
- [ ] Use `valgrind --tool=massif` to profile heap usage over time; view the report with `ms_print`.
- [ ] Use `valgrind --tool=helgrind` to detect data races in a multithreaded program; fix the race and confirm a clean report.
- [ ] Explain the performance overhead of Valgrind (10–50× slower); describe when to use Valgrind vs ASan.
- [ ] Run Valgrind on a program with an invalid stack read (off-by-one on a stack buffer); observe the `Invalid read` report.

### 8.3 AddressSanitizer (ASan)

- [ ] Compile with `-fsanitize=address,leak -g`; trigger a heap buffer overflow and read the ASan report's shadow memory dump.
- [ ] Compile with `-fsanitize=undefined` (UBSan); trigger signed integer overflow, null dereference, and misaligned access; read each report.
- [ ] Compile with `-fsanitize=thread` (TSan); trigger a data race; read the race report and fix the bug.
- [ ] Explain the shadow memory concept ASan uses: every 8 bytes of real memory are tracked by 1 byte of shadow memory.
- [ ] Combine ASan with GDB: set `ASAN_OPTIONS=abort_on_error=1` and run under GDB to get a backtrace at the exact error.

### 8.4 objdump

- [ ] Disassemble a binary with `objdump -d -M intel ./program` (Intel syntax) and with `objdump -d ./program` (AT&T syntax); explain syntax differences.
- [ ] Use `objdump -t` to list all symbols; use `-T` for dynamic symbols; use `-r` to list relocation entries.
- [ ] Use `objdump -s -j .rodata` to dump the read-only data section and find hardcoded strings.
- [ ] Use `objdump -p` to print private headers including the list of shared library dependencies.

### 8.5 readelf

- [ ] Use `readelf -a ./binary` for a full dump; then use `-h`, `-S`, `-l`, `-s`, `-d` individually and describe what each shows.
- [ ] Use `readelf -n` to read the `.note.gnu.build-id` section; explain what a build ID is used for.
- [ ] Use `readelf --debug-dump=info` to inspect DWARF debug information embedded in a binary compiled with `-g`.

### 8.6 nm, strings, ldd

- [ ] Use `nm ./binary | sort` to list symbols by address; use `nm -u` to list only undefined (externally resolved) symbols.
- [ ] Use `nm -D` on a shared library to list its exported dynamic symbols.
- [ ] Use `strings -n 8 ./binary` to print strings of minimum length 8; redirect to a file and search for interesting patterns.
- [ ] Use `ldd ./binary` to list shared library dependencies; use `ldd -v` to see symbol version information.
- [ ] Use `LD_DEBUG=libs ./binary` to trace the dynamic linker's library search path at runtime.

### 8.7 strace and ltrace

- [ ] Use `strace ./program` to trace all system calls; use `-e trace=file` to filter only file-related calls; use `-e trace=network` for network calls.
- [ ] Use `strace -p <PID>` to attach to a running process (one you own) and observe its system calls live.
- [ ] Use `strace -c ./program` to count system calls and measure time spent in each.
- [ ] Use `ltrace ./program` to trace library function calls; compare the output with `strace` for the same run.
- [ ] Use `strace` to trace a `fork`+`exec` sequence; use `-f` to follow child processes.

### 8.8 checksec

- [ ] Run `checksec --file=./binary` on several binaries and interpret every field: `RELRO`, `Stack`, `NX`, `PIE`, `FORTIFY`, `ASAN`, `CFI`.
- [ ] Compile the same source four times with different hardening flags; run `checksec` after each and compare the output.
- [ ] Use `checksec --proc=all` to check mitigations on all running processes (requires root or appropriate permissions in a VM).
- [ ] Explain what each mitigation prevents and its cost: Full RELRO (GOT overwrite), Stack Canary (sequential stack overflow), NX (shellcode execution), PIE (fixed-address ROP), FORTIFY (unsafe libc calls).

---

## 9. Automation and Mini Projects

### 9.1 TCP Client/Server

- [ ] Build a TCP echo server in C: `socket` → `bind` → `listen` → `accept` → `recv`/`send` loop → `close`; handle `SIGINT` for clean shutdown.
- [ ] Build a matching TCP client that connects, sends a line of text, receives the echo, and disconnects cleanly.
- [ ] Extend the server to support multiple clients with `select`/`poll`; explain why this avoids spawning a thread per client.
- [ ] Add a simple text protocol: the server responds to `ECHO <msg>`, `TIME`, and `QUIT`; document the protocol in a comment.
- [ ] Handle partial `send`/`recv` by looping until all bytes are transferred; explain why a single `recv` may not return all requested bytes.

### 9.2 Simple Shell

- [ ] Build a shell that reads a line with `fgets`, tokenizes it with `strtok`, `fork`s, calls `execvp` in the child, and `waitpid`s in the parent.
- [ ] Add support for built-in commands: `cd` (using `chdir`), `exit`, `pwd` (using `getcwd`).
- [ ] Add output redirection (`> file`) by opening the file and using `dup2` to replace stdout before `exec`.
- [ ] Add a pipe operator (`|`) by creating a `pipe()`, forking twice, and connecting the write end to the left command's stdout and the read end to the right command's stdin.
- [ ] Add basic signal handling: ignore `SIGINT` in the shell itself but allow child processes to receive it.
- [ ] Add a command history buffer (ring buffer of last N commands) accessible with `!N` or `!!`.

### 9.3 Mini HTTP Server

- [ ] Build an HTTP/1.0 server that accepts a TCP connection, reads the first line of the HTTP request, and serves a static HTML file from a fixed directory.
- [ ] Parse the `GET /path HTTP/1.0` request line; extract the path; validate it stays inside the document root (prevent path traversal).
- [ ] Return correct status lines: `200 OK` for found files, `404 Not Found` for missing ones, `403 Forbidden` for directories or permission-denied paths.
- [ ] Set the `Content-Type` header based on the file extension (`.html`, `.css`, `.js`, `.txt`).
- [ ] Handle multiple requests sequentially first; then extend to use `fork` for concurrent clients.

### 9.4 File Parser

- [ ] Build a CSV parser in C: read a file line by line with `fgets`; split each line on commas; store records in a dynamically grown array of structs.
- [ ] Build a binary file parser: read a fixed-size header struct with `fread`; validate magic bytes; parse and print fields.
- [ ] Write a log analyzer: scan a log file line by line with `getline`; use `sscanf` to extract timestamp, level, and message; count each log level.
- [ ] Build an INI-file parser: read `[section]` headers and `key = value` pairs; store them in a nested struct or a flat array.

### 9.5 Process Monitor

- [ ] List all running processes by reading `/proc` entries with `opendir`/`readdir`; for each numeric directory (PID), read `/proc/<pid>/status` and print name and state.
- [ ] Track CPU and memory usage for a process by reading `/proc/<pid>/stat` and `/proc/<pid>/statm` repeatedly at a fixed interval.
- [ ] Watch for a process to start (poll `/proc` until a matching name appears) and print its PID and start time.
- [ ] Print the full command line of a process from `/proc/<pid>/cmdline` (NUL-delimited fields).

### 9.6 Memory Allocator (Mini malloc)

- [ ] Implement a simple arena (linear) allocator: one large `mmap`-ed block; a bump pointer for allocation; no individual `free` — only a bulk `reset`.
- [ ] Implement a free-list allocator: add a `Header` struct before each allocation; maintain a singly linked list of free blocks; implement `my_malloc`, `my_free`, and first-fit search.
- [ ] Add coalescing: when `my_free` is called, merge the freed block with adjacent free blocks to reduce fragmentation.
- [ ] Test your allocator: run it against a sequence of allocations and frees; compare its behaviour with glibc `malloc` using Valgrind.
- [ ] Add a debug mode that prints all allocated and free blocks on request.

### 9.7 ELF Parser

- [ ] Read an ELF binary from disk; parse the ELF header (`Elf64_Ehdr`); print magic, class, data encoding, type, machine, and entry point.
- [ ] Parse the section header table; print each section's name (from `.shstrtab`), type, address, offset, and size.
- [ ] Parse the program header table; print each segment's type, flags, virtual address, file size, and memory size.
- [ ] Extract and print all symbol names from the `.symtab` or `.dynsym` section.
- [ ] Find and print all string literals in the `.rodata` section by walking the section content byte by byte.

### 9.8 Simple Debugger

- [ ] Use `ptrace(PTRACE_TRACEME, ...)` in a child and `ptrace(PTRACE_ATTACH, ...)` in the parent to attach to a process.
- [ ] Implement software breakpoints: replace a byte at a target address with `0xCC` (INT 3) using `PTRACE_POKETEXT`; restore it on hit.
- [ ] After a `SIGTRAP`, use `PTRACE_GETREGS` to read the registers; print the instruction pointer and all general-purpose registers.
- [ ] Implement single-step execution using `PTRACE_SINGLESTEP`; print the instruction pointer after each step.
- [ ] Use `PTRACE_PEEKDATA` to read memory from the traced process and print it as hex bytes.

### 9.9 Binary File Analyser

- [ ] Build a tool that accepts a file path; detects its type from magic bytes (ELF, PE, ZIP, PDF, PNG, JPEG); prints a human-readable report.
- [ ] For ELF files: print the architecture, entry point, all section names and sizes, and all imported symbols.
- [ ] Use `strings` logic internally: scan for runs of printable ASCII bytes of minimum length N and print them with their file offsets.
- [ ] Compute and print the MD5 and SHA-256 hash of the file using `<openssl/md5.h>` or a pure-C implementation.
- [ ] Print an entropy analysis: divide the file into 256-byte blocks; compute the byte-frequency entropy of each block; flag high-entropy regions as possibly compressed or encrypted.

---

## 10. Learning Outcome

- [ ] Write complete, correct C programs covering all syntax and language features without referencing documentation.
- [ ] Explain the memory layout of any C variable, struct, or pointer — stack, heap, `.data`, or `.bss` — without hesitation.
- [ ] Read a GDB backtrace and identify the crashing function, its arguments, and the corrupted memory region.
- [ ] Use `objdump -d`, `readelf`, `nm`, and `strings` to extract information from an ELF binary without running it.
- [ ] Explain every entry in `checksec` output and describe how each mitigation raises the bar for exploitation.
- [ ] Write a multi-process or multi-threaded C program using `fork`/`pthread`, pipes, shared memory, and semaphores correctly.
- [ ] Build a working TCP client/server, a simple shell, and a basic file parser from scratch without copying templates.
- [ ] Identify at least five classes of memory vulnerability (buffer overflow, UAF, double-free, format string, integer overflow) by reading C code.
- [ ] Use Valgrind and ASan together to achieve zero reported memory errors on every project you write.
- [ ] Explain the full lifecycle of a Linux process: `fork` → `exec` → memory layout setup → signal handling → `exit` → zombie collection.
- [ ] Understand enough about binary structure and memory corruption to read CTF writeups and introductory exploit-development resources fluently.
