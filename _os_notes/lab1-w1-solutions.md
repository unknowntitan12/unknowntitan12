---
layout: page
title: "Lab1-w1: sleep & memdump"
short_title: "W1: sleep & memdump"
lab: lab1
week: 1
order: 1
date: 2026-01-06
---

<div class="info-box">
📦 <strong>Lab zip:</strong> xv6labs-w1 &nbsp;|&nbsp; 📊 <strong>Score:</strong> 40/40 &nbsp;|&nbsp; 📅 <strong>Due:</strong> Week 1
</div>

## Task 1 — `sleep`

### What you need to do

Implement a user-level `sleep` program that calls the xv6 `pause()` system call to pause for a specified number of ticks.

**File:** `user/sleep.c`  
**Add to Makefile:** `$U/_sleep\` under `UPROGS`

### Key points

- Convert the command-line argument string to an integer using `atoi()`
- Call `pause(n)` — not the UNIX `sleep()` — to block for `n` kernel ticks
- Print an error and exit with code 1 if no argument is given
- Call `exit(0)` at the end

### Solution

```c
// user/sleep.c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    if (argc != 2) {
        fprintf(2, "Usage: sleep <ticks>\n");
        exit(1);
    }
    int ticks = atoi(argv[1]);
    pause(ticks);
    exit(0);
}
```

### Makefile entry

Add this line inside the `UPROGS` block:

```makefile
$U/_sleep\
```

### Running the tests

```bash
./grade-lab-util sleep
# == Test sleep, no arguments == sleep, no arguments: OK
# == Test sleep, returns      == sleep, returns: OK
# == Test sleep, makes syscall == sleep, makes syscall: OK
```

---

## Task 2 — `memdump`

### What you need to do

Implement `memdump(char *fmt, char *data)` — a function that interprets a format string to print successive bytes from a raw memory buffer.

**File:** `user/memdump.c` (fill in the function body)

### Format characters

| Char | Bytes consumed | Output |
|------|---------------|--------|
| `i`  | 4 | 32-bit int, decimal |
| `p`  | 8 | 64-bit int, hex |
| `h`  | 2 | 16-bit int, decimal |
| `c`  | 1 | 8-bit ASCII char |
| `s`  | 8 | dereference 64-bit pointer → print C string |
| `S`  | rest | print bytes as null-terminated C string |

### Key concepts

- Use a `char *ptr` that advances through `data` as you consume bytes
- Cast `ptr` to the right integer pointer type, dereference, then advance `ptr`
- For `s`: cast to `char **`, dereference to get the pointer, then `printf` the pointed-to string
- For `S`: just `printf("%s", ptr)` — xv6's printf handles null-terminated strings correctly

### Solution

```c
void
memdump(char *fmt, char *data)
{
    char *ptr = data;

    for (char *f = fmt; *f; f++) {
        switch (*f) {
        case 'i': {
            // 4 bytes → 32-bit signed int, decimal
            int val = *(int *)ptr;
            printf("%d\n", val);
            ptr += 4;
            break;
        }
        case 'p': {
            // 8 bytes → 64-bit unsigned, hex
            uint64 val = *(uint64 *)ptr;
            printf("%lx\n", val);
            ptr += 8;
            break;
        }
        case 'h': {
            // 2 bytes → 16-bit unsigned, decimal
            uint16 val = *(uint16 *)ptr;
            printf("%d\n", val);
            ptr += 2;
            break;
        }
        case 'c': {
            // 1 byte → ASCII char
            char val = *ptr;
            printf("%c\n", val);
            ptr += 1;
            break;
        }
        case 's': {
            // 8 bytes → pointer to C string; print the string
            char *str = *(char **)ptr;
            printf("%s\n", str);
            ptr += 8;
            break;
        }
        case 'S': {
            // rest of data → null-terminated C string
            printf("%s\n", ptr);
            return;   // 'S' consumes the rest
        }
        default:
            break;
        }
    }
}
```

### Why `uint16` works

xv6 defines `uint16` in `kernel/types.h`. If the build complains, use `unsigned short` instead — both are 2 bytes on RISC-V.

### Running the tests

```bash
./grade-lab-util memdump
# == Test memdump, examples          == memdump, examples: OK
# == Test memdump, format ii, S, p   == memdump, format ii, S, p: OK
```

---

## Variation Questions

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

Modify `sleep` so that it accepts an **optional** second argument `unit` which can be `"ticks"` or `"ms"`. If the unit is `"ms"`, convert milliseconds to ticks (assume 100 ticks/second). If no unit is given, default to ticks.

Example: `sleep 500 ms` should pause for 50 ticks.

<div class="answer-content">

<pre><code class="language-c">int
main(int argc, char *argv[])
{
    if (argc &lt; 2 || argc &gt; 3) {
        fprintf(2, "Usage: sleep &lt;n&gt; [ticks|ms]\n");
        exit(1);
    }
    int n = atoi(argv[1]);
    if (argc == 3 &amp;&amp; strcmp(argv[2], "ms") == 0) {
        // 100 ticks per second → 1 ms = 0.1 ticks
        // integer arithmetic: n ms / 10 = ticks
        n = n / 10;
        if (n &lt; 1) n = 1;
    }
    pause(n);
    exit(0);
}
</code></pre>

<p>The key insight: ticks are coarser than milliseconds, so you divide, not multiply.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-med">Medium</span></div>

Add format character `'b'` to `memdump` that prints the **next byte in binary** (8 bits), e.g. the byte `0x41` ('A') should print `01000001`.

<div class="answer-content">

<pre><code class="language-c">case 'b': {
    unsigned char byte = *(unsigned char *)ptr;
    for (int i = 7; i &gt;= 0; i--) {
        printf("%d", (byte &gt;&gt; i) &amp; 1);
    }
    printf("\n");
    ptr += 1;
    break;
}
</code></pre>

<p>The loop starts from bit 7 (MSB) down to bit 0 (LSB). <code>(byte >> i) & 1</code> extracts each bit.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-med">Medium</span></div>

Add format character `'a'` to `memdump` that prints the **next N bytes as a hex dump** (address offset + hex + ASCII), where N is given as the next character in the format string (a single decimal digit). For example, `"a4"` should hex-dump 4 bytes.

<div class="answer-content">

<pre><code class="language-c">case 'a': {
    f++;                         // consume the digit after 'a'
    if (*f &lt; '1' || *f &gt; '9') break;
    int n = *f - '0';
    for (int i = 0; i &lt; n; i++) {
        printf("%02x ", (unsigned char)ptr[i]);
    }
    printf("  ");
    for (int i = 0; i &lt; n; i++) {
        char c = ptr[i];
        printf("%c", (c &gt;= 32 &amp;&amp; c &lt; 127) ? c : '.');
    }
    printf("\n");
    ptr += n;
    break;
}
</code></pre>

<p>Key: advance <code>f</code> an extra step to read the digit, and use <code>%02x</code> for zero-padded hex.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-hard">Hard</span></div>

Extend `memdump` to handle format character `'r'` which repeats the *previous* format character N times, where N is an integer that follows in the format string (e.g. `"ir3"` means print one `i` then repeat the previous character `r` 3 times = total 4 ints).

<div class="answer-content">

<p>The key trick is to remember the previous character and re-process it.</p>

<pre><code class="language-c">void
memdump(char *fmt, char *data)
{
    char *ptr = data;
    char prev = 0;

<p>for (char *f = fmt; *f; f++) {         char cur = *f;</p>

<p>if (cur == 'r') {             // parse integer count that follows             f++;             int count = 0;             while (*f &gt;= '0' &amp;&amp; *f &lt;= '9') {                 count = count * 10 + (*f - '0');                 f++;             }             f--;  // back up so outer loop increment is correct             // repeat prev 'count' times             char tmp[2] = { prev, 0 };             for (int i = 0; i &lt; count; i++)                 memdump(tmp, ptr + /* offset depends on prev type */0);             // simpler: inline the switch with prev:             // (omitted for brevity — use the same switch logic, just loop)         } else {             prev = cur;             // ... normal switch on cur ...         }     } } </code></pre></p>

<p>A cleaner approach: factor the single-character dispatch into a helper function <code>process_char(char c, char **ptr)</code> that returns bytes consumed, then the <code>'r'</code> case can call it N times.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-easy">Easy</span></div>

What happens if you call `sleep` with a negative number like `sleep -5`? Explain the behavior and suggest a fix.

<div class="answer-content">

<code>atoi("-5")</code> returns <code>-5</code>. When <code>pause(-5)</code> is called, the xv6 kernel implementation of <code>sys_pause</code> likely casts it to an unsigned integer or checks <code><= 0</code> and returns immediately, so <code>sleep -5</code> effectively sleeps for 0 ticks.

<strong>Fix:</strong> Add a guard in <code>sleep.c</code>:

<pre><code class="language-c">int ticks = atoi(argv[1]);
if (ticks &lt; 0) {
    fprintf(2, "sleep: ticks cannot be negative\n");
    exit(1);
}
pause(ticks);
</code></pre>

</div>
</div>
