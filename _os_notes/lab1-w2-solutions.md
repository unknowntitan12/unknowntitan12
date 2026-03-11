---
layout: page
title: "Lab1-w2: hello, sixfive & xargs"
short_title: "W2: hello, sixfive & xargs"
lab: lab1
week: 2
order: 2
date: 2026-01-15
---

<div class="info-box">
📦 <strong>Lab zip:</strong> xv6labs-w2 &nbsp;|&nbsp; 📊 <strong>Score:</strong> 55/55 &nbsp;|&nbsp; 📅 <strong>Due:</strong> Week 2
</div>

## Task 1 — `hello` (Custom Kernel Syscall)

### What you need to do

Add a new `hello()` system call that prints two lines from kernel space, then create a user program to call it.

### All files to modify

| File | What to add |
|------|------------|
| `kernel/syscall.h` | `#define SYS_hello 23` |
| `kernel/syscall.c` | `extern` declaration + table entry |
| `kernel/sysproc.c` | `sys_hello()` implementation |
| `kernel/defs.h` | prototype `uint64 sys_hello(void);` |
| `user/usys.pl` | `entry("hello");` |
| `user/user.h` | `int hello(void);` |
| `user/hello.c` | new user program |
| `Makefile` | `$U/_hello\` |

### Kernel handler — `kernel/sysproc.c`

```c
uint64
sys_hello(void)
{
    printf("Hello from kernel syscall!\n");
    printf("I am ICT1012!\n");
    return 0;
}
```

### Syscall table — `kernel/syscall.h` (add at end)

```c
#define SYS_hello  23
```

### Syscall table — `kernel/syscall.c` (add two lines)

```c
extern uint64 sys_hello(void);          // with other externs

// inside syscalls[] array:
[SYS_hello]  sys_hello,
```

### User stub — `user/usys.pl`

```perl
entry("hello");
```

### User header — `user/user.h`

```c
int hello(void);
```

### User program — `user/hello.c`

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(void)
{
    int r = hello();
    printf("hello returned %d\n", r);
    return 0;
}
```

### Expected output

```
$ hello
Hello from kernel syscall!
I am ICT1012!
hello returned 0
```

---

## Task 2 — `sixfive`

### What you need to do

Read one or more files and print every number that is a **multiple of 5 or 6**. Numbers are sequences of decimal digits separated by the characters `" -\r\t\n./,"`. The string `"xv6"` contains the literal `6` preceded by `v` — that **does not** count as a standalone number.

**File:** `user/sixfive.c`

### Algorithm

1. Read the file one character at a time
2. When you hit a digit, start building a number
3. When you hit a separator (or EOF), check if the accumulated number is divisible by 5 or 6

### Solution

```c
// user/sixfive.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fcntl.h"

#define SEPS " -\r\t\n./,"

// Returns 1 if n is a multiple of 5 or 6
int is_sixfive(int n) {
    return (n % 5 == 0 || n % 6 == 0);
}

void process(int fd) {
    char c;
    int in_num = 0;
    int num = 0;

    while (read(fd, &c, 1) > 0) {
        if (c >= '0' && c <= '9') {
            in_num = 1;
            num = num * 10 + (c - '0');
        } else if (strchr(SEPS, c)) {
            if (in_num && is_sixfive(num)) {
                printf("%d\n", num);
            }
            in_num = 0;
            num = 0;
        } else {
            // non-digit, non-separator: end any number in progress
            if (in_num && is_sixfive(num)) {
                printf("%d\n", num);
            }
            in_num = 0;
            num = 0;
        }
    }
    // flush end of file
    if (in_num && is_sixfive(num)) {
        printf("%d\n", num);
    }
}

int
main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(2, "Usage: sixfive file...\n");
        exit(1);
    }
    for (int i = 1; i < argc; i++) {
        int fd = open(argv[i], O_RDONLY);
        if (fd < 0) {
            fprintf(2, "sixfive: cannot open %s\n", argv[i]);
            exit(1);
        }
        process(fd);
        close(fd);
    }
    exit(0);
}
```

### Important edge cases

- The word `xv6` — the `6` is surrounded by non-separator, non-digit characters, so it **does** trigger "end of number" at `v` and starts a new run from `6`. Actually: `x` is non-digit non-separator → flush; `v` is non-digit non-separator → flush; `6` starts a number; then any separator/EOF would print 6. The spec says "for the six in xv6, sixfive shouldn't print 6 **but for `/6,` it should**". So `xv6` means `6` follows `v` which is a non-separator — the number should still flush when we hit the next separator. The trick: when in `xv6`, after `6` comes end-of-token (e.g. newline) which **is** a separator, so `6` would normally be printed. But the grader test `sixfive_readme` runs `sixfive README` on the xv6 README which contains `"xv6"` — and 6 should **not** be printed. The resolution: a non-separator non-digit character **before** a digit starts the number invalidates that number.

The cleaner approach the graders expect:

```c
// Only enter number-collection mode if the previous character
// was a separator (or start-of-file)
void process(int fd) {
    char c;
    int num = 0;
    int in_num = 0;
    int after_sep = 1;   // treat start-of-file as separator

    while (read(fd, &c, 1) > 0) {
        if (c >= '0' && c <= '9') {
            if (after_sep || in_num) {
                in_num = 1;
                num = num * 10 + (c - '0');
            }
            after_sep = 0;
        } else if (strchr(SEPS, c)) {
            if (in_num && is_sixfive(num))
                printf("%d\n", num);
            in_num = 0;
            num = 0;
            after_sep = 1;
        } else {
            // non-digit, non-separator: cancel
            if (in_num && is_sixfive(num))
                printf("%d\n", num);
            in_num = 0;
            num = 0;
            after_sep = 0;
        }
    }
    if (in_num && is_sixfive(num))
        printf("%d\n", num);
}
```

---

## Task 3 — `xargs`

### What you need to do

Read lines from stdin; for each line, run the command given as arguments with that line appended.

**File:** `user/xargs.c`

### Algorithm

1. Parse `argv` to get the base command (e.g. `echo bye`)
2. Read stdin one character at a time; collect into a buffer until `'\n'`
3. On newline: `fork()`, then in child: `exec()` the command with the line appended
4. Parent `wait()`s for child to finish
5. Repeat until EOF

### Solution

```c
// user/xargs.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/param.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(2, "Usage: xargs <command> [args...]\n");
        exit(1);
    }

    char buf[512];
    char *xargv[MAXARG];
    int n;

    // copy the base command into xargv
    for (n = 1; n < argc; n++)
        xargv[n - 1] = argv[n];
    // xargv[argc-1] will be the line from stdin
    // xargv[argc]   will be 0 (null terminator)

    char line[512];
    int pos = 0;
    char c;

    while (read(0, &c, 1) > 0) {
        if (c == '\n') {
            line[pos] = '\0';
            pos = 0;

            xargv[argc - 1] = line;
            xargv[argc]     = 0;

            int pid = fork();
            if (pid == 0) {
                exec(xargv[0], xargv);
                fprintf(2, "xargs: exec failed\n");
                exit(1);
            }
            wait(0);
        } else {
            if (pos < (int)sizeof(line) - 1)
                line[pos++] = c;
        }
    }

    // handle last line without trailing newline
    if (pos > 0) {
        line[pos] = '\0';
        xargv[argc - 1] = line;
        xargv[argc]     = 0;
        int pid = fork();
        if (pid == 0) {
            exec(xargv[0], xargv);
            exit(1);
        }
        wait(0);
    }

    exit(0);
}
```

### Understanding `xargstest.sh`

The test runs:
```sh
echo hello
echo hello
echo hello
```
piped through xargs — each `hello` line gets appended to the base command. The many `$` prompts appear because the xv6 shell still prints a prompt for each command read from the file.

---

## Variation Questions

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

Instead of printing `"Hello from kernel syscall!"`, modify `sys_hello` to accept one integer argument (the caller's PID passed explicitly) and print `"Hello from PID <n>!"`. What xv6 kernel function do you use to read an integer argument inside a syscall handler?

<div class="answer-content">

<p>Use <code>argint(0, &pid)</code> to read the first integer argument:</p>

<pre><code class="language-c">uint64
sys_hello(void)
{
    int pid;
    argint(0, &amp;pid);
    printf("Hello from PID %d!\n", pid);
    return 0;
}
</code></pre>

<p>In <code>user/hello.c</code>, pass <code>getpid()</code>:</p>

<pre><code class="language-c">hello(getpid());
</code></pre>

<p>And update the prototype in <code>user/user.h</code>:</p>

<pre><code class="language-c">int hello(int pid);
</code></pre>

<code>argint(n, &dest)</code> reads the n-th argument from the user's trap frame registers (a0, a1, a2...). The first argument is in register a0.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-med">Medium</span></div>

Modify `sixfive` to print numbers that are multiples of **any divisor passed as a command-line flag**. For example `sixfive -d 7 file.txt` prints multiples of 7. If `-d` is not given, default to printing multiples of 5 or 6.

<div class="answer-content">

<pre><code class="language-c">int
main(int argc, char *argv[])
{
    int divisors[16];
    int ndiv = 0;
    int i = 1;

<p>// parse -d flags     while (i &lt; argc &amp;&amp; argv[i][0] == '-') {         if (strcmp(argv[i], "-d") == 0 &amp;&amp; i + 1 &lt; argc) {             divisors[ndiv++] = atoi(argv[++i]);         }         i++;     }</p>

<p>// defaults     if (ndiv == 0) {         divisors[ndiv++] = 5;         divisors[ndiv++] = 6;     }</p>

<p>// process remaining args as files     for (; i &lt; argc; i++) {         int fd = open(argv[i], O_RDONLY);         if (fd &lt; 0) { fprintf(2, "sixfive: cannot open %s\n", argv[i]); exit(1); }         process(fd, divisors, ndiv);         close(fd);     }     exit(0); } </code></pre></p>

<p>Update <code>is_sixfive</code> → <code>is_match(int n, int *divs, int ndiv)</code> that loops through <code>divs</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-med">Medium</span></div>

Modify `xargs` to support a `-n <max>` flag (like the real xargs) that collects up to `<max>` lines and passes them all as separate arguments to a single invocation of the command. For example, `echo -e "a\nb\nc" | xargs -n 2 echo` should produce:

```
a b
c
```

<div class="answer-content">

<p>The key change: buffer up to <code>max</code> lines, then pass them all as arguments in one exec call.</p>

<pre><code class="language-c">int max = 1;   // default: 1 line per invocation

<p>// parse -n flag before processing argv int base = 1; if (argc &gt;= 3 &amp;&amp; strcmp(argv[1], "-n") == 0) {     max = atoi(argv[2]);     base = 3; }</p>

<p>// collect lines into extra_args[] char *extra[MAXARG]; int nextra = 0; // ... read lines, push into extra[], when nextra == max → exec, reset nextra </code></pre></p>

<p>On exec: <pre><code class="language-c">// build xargv = base_command + extra[0..nextra-1] int k = argc - base; for (int j = 0; j &lt; k; j++) xargv[j] = argv[base + j]; for (int j = 0; j &lt; nextra; j++) xargv[k + j] = extra[j]; xargv[k + nextra] = 0; </code></pre></p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-hard">Hard</span></div>

Implement a `syscount` system call that returns the total number of system calls made by the calling process since it started. Where would you store the per-process counter in the xv6 kernel, and how would it be initialized?

<div class="answer-content">

<strong>Step 1:</strong> Add <code>int syscall_count;</code> to <code>struct proc</code> in <code>kernel/proc.h</code>.

<strong>Step 2:</strong> Initialize it in <code>allocproc()</code> in <code>kernel/proc.c</code>:

<pre><code class="language-c">p-&gt;syscall_count = 0;
</code></pre>

<strong>Step 3:</strong> Increment it in <code>syscall()</code> in <code>kernel/syscall.c</code>:

<pre><code class="language-c">p-&gt;syscall_count++;
</code></pre>

<strong>Step 4:</strong> Implement <code>sys_syscount()</code> in <code>kernel/sysproc.c</code>:

<pre><code class="language-c">uint64
sys_syscount(void)
{
    struct proc *p = myproc();
    return p-&gt;syscall_count;
}
</code></pre>

<p>Register it with the usual <code>syscall.h</code> / <code>syscall.c</code> / <code>user/user.h</code> / <code>usys.pl</code> steps.</p>

<p>The counter is stored per-process in the <code>proc</code> struct because each process has independent lifetime. It's initialized to 0 in <code>allocproc()</code> which runs when the process table slot is first allocated.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-hard">Hard</span></div>

Explain why the `exec()` call in `xargs`'s child does **not** need to `free()` the `line` buffer before calling `exec`, even though `line` was allocated (on the stack) by the parent process.

<div class="answer-content">

<p>When <code>exec()</code> succeeds, it <strong>replaces</strong> the entire address space of the current process with the new program. The old stack, heap, and data segments are discarded entirely. There is no "memory leak" because the OS reclaims every page of the old address space as part of the exec system call.</p>

<p>If <code>exec()</code> fails, the process continues running with the original address space (xv6's <code>exec</code> only replaces the address space after it has fully loaded the new program), so <code>line</code> is still valid. We then call <code>exit(1)</code> anyway, which also reclaims all memory.</p>

<p>This is a fundamental OS guarantee: <code>exec</code> and <code>exit</code> always clean up — you never need to free memory before either call.</p>

</div>
</div>
