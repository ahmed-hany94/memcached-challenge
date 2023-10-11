# memcached interview question

## Table of Contents:
- [What is it](#what-is-it)
- [Why](#why)
- [How](#how)
- [Solution](#solution)
- [Credits](#credits)

## What is it
### `Add a mult command to memcached.`

```bash
mult age 10          # input
380                  # output
```

## Why
This question tests the tasks that you deal with on a day to day basis working in a software company.

It tests: building source code; reading other people's source code; modifying other people's source code.

## How
The release that was used in [The best engineering interview question I’ve ever gotten, Part 1](https://quuxplusone.github.io/blog/2022/01/06/memcached-interview/) is `1.4.15`, while the current release of `memcached` is `1.6.21`. I will use the one in the blog. I will try to not consult the writeup as much as I can and document the process.

I am using wsl2 on windows 11.

1- Download the source code and install it.
```
$ curl -O https://memcached.org/files/memcached-1.4.15.tar.gz
$ tar zxvf memcached-1.4.15.tar.gz
$ cd memcached-1.4.15
$ ./configure
$ make -j3
```

I removed `-Werror` from the `Makefile` to be able to compile. This creates an elf binary `memcached`.

2- run memcached. & connect to it.

```bash
$ ./memcached
```

It just runs with no output. 

In another terminal
```
$ telnet 127.0.0.1 11211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
```

We are connected.

3- store a value & retrieve it.
```bash
set fullname 0 25 10 # input
John Smith           # input
STORED               # output
```
We set a string key called `fullname` with a `10` byte value `John Smith`, flags `0` and expiry timeout `25` seconds.

```bash
get fullname         # input
VALUE fullname 0 10  # output
John Smith           # output
END                  # output
``` 
We retrieve the value of our key with its byte size and flag.

4- Running `cloc` on it.

```
$ cloc .

155 text files.
148 unique files.         
50 files ignored.

github.com/AlDanial/cloc v 1.90  T=0.08 s (1329.1 files/s, 485797.0 lines/s)

---------------------------------------------
Language           files blank comment  code
---------------------------------------------
Bourne Shell         8   1168   1564    9975
C                   15   1499   1135    8734
Perl                51    965    446    3809
XML                  3    114      1    1594
make                 4    212    203    1488
m4                   4    183     76    1483
Bourne Again Shell   2    106    175    1050
C/C++ Header        12    219    447     882
XSLT                 2     23      2     162
DTD                  1     69     74     161
D                    1     32    235      39
Markdown             1     16      0      22
YAML                 1      0      0      16
---------------------------------------------
SUM:               105   4606   4358   29415
---------------------------------------------
```

5- The trick that was used by `tsoding` and which I have usually used before when diving into a new code base especially if the code is written in a simple manner with a few abstractions is to search for some functionality presented to the user and get your bearings from there. This might be preceded by finding the entry point of the program first but for my purposes here, I will skip that step.

I searched for command (or grep) for `flush_all` and found several matches, most intriguing is in `memcached.c`. line `3288`.
```c
else if (ntokens >= 2 && ntokens <= 4 && (strcmp(tokens[COMMAND_TOKEN].value, "flush_all") == 0)) {
    ...
}
```

This is a part of `process_command` function which seemingly contains the processing of all the other memcached commands.

6- So the challenge mentions `incr` && `decr` as two operations that perform arithmetic operations on values and the `mult` operation should follow suit. Well the `incr` operation. I used [`gf`](https://github.com/nakst/gf) (a `gdb` frontend) to attach to the running `memcached` instance using `pgrep` to get the `PID` of the parent process. 

Here is how `incr`/`decr` gets processed.

```c
// incr
else if ((ntokens == 4 || ntokens == 5) && (strcmp(tokens[COMMAND_TOKEN].value, "incr") == 0)) {
    process_arithmetic_command(c, tokens, ntokens, 1);
} 

// decr
else if ((ntokens == 4 || ntokens == 5) && (strcmp(tokens[COMMAND_TOKEN].value, "decr") == 0)) {
    process_arithmetic_command(c, tokens, ntokens, 1);
}
```

Located in `memcached.c` at line `2992`, the function `process_arithmetic_command` calls `add_delta` at line `576` in `thread.c` which itself calls `do_add_delta` back in `memcached.c` at line `3052`.

The `do_add_delta` which does the actual arithmetic work has the signature
```c
delta_result_type do_add_delta(
    conn *c,
    const char *key, 
    const size_t nkey,
    const <error-type> incr, // 1 to incr, 0 to decr
    const int64_t delta, 
    char *buf, 
    uint64_t *cas, 
    const uint32_t hv
){}
```
## Solution

### First Pass Hacky Solution:

The `incr`/`decr` functionality are determined by a boolean value being passed to the `process_arithmetic_command` function all the way to the `do_add_delta` function. So what's a step up from a boolean (binary value)? My first instinct is an enum.

```c
enum arithmetic_op {
    OP_INCR,
    OP_DECR,
    OP_MULT,
    OP_CNT
};
```

Then it's all about fixing all the side-effects that my change will introduce.

First are the calling functions in `process_command`

```c
// incr
else if ((ntokens == 4 || ntokens == 5) && (strcmp(tokens[COMMAND_TOKEN].value, "incr") == 0)) {
    process_arithmetic_command(c, tokens, ntokens, OP_INCR);
} 

// decr
else if ((ntokens == 4 || ntokens == 5) && (strcmp(tokens[COMMAND_TOKEN].value, "decr") == 0)) {
    process_arithmetic_command(c, tokens, ntokens, OP_DECR);
}

// mult
else if (ntokens == 4 && (strcmp(tokens[COMMAND_TOKEN].value, "decr") == 0)) {
    process_arithmetic_command(c, tokens, ntokens, OP_MULT);
}
```

In `process_arithmetic_command`, we change the signature of the function to accept `op` instead of a boolean `incr`.

```c
static void process_arithmetic_command(conn *c, token_t *tokens, const size_t ntokens, const size_t op)
```

We pass it to `add_delta`,

```c
add_delta(c, key, nkey, op, delta, temp, NULL)
```

which pass it to `do_add_delta`.

```c
do_add_delta(c, key, nkey, op, delta, buf, cas, hv);
```

We'll again change the signature of `do_add_delta` in both `memcached.h` & `memcached.c` 


```c
enum delta_result_type do_add_delta(conn *c, const char *key, const size_t nkey,
                                    const int op, const int64_t delta,
                                    char *buf, uint64_t *cas,
                                    const uint32_t hv);

```

Then we change the logic of the function to actually do what we want.

```c
if (op == OP_INCR) {
        value += delta;
        MEMCACHED_COMMAND_INCR(c->sfd, ITEM_key(it), it->nkey, value);
} else if(op == OP_DECR) {
    if(delta > value) {
        value = 0;
    } else {
        value -= delta;
    }
    MEMCACHED_COMMAND_DECR(c->sfd, ITEM_key(it), it->nkey, value);
} else if(op == OP_MULT) {
    value *= delta;
    MEMCACHED_COMMAND_DECR(c->sfd, ITEM_key(it), it->nkey, value);
}
```

and also add to updating the stats like before.

```c
pthread_mutex_lock(&c->thread->stats.mutex);
if (op == OP_INCR) {
    c->thread->stats.slab_stats[it->slabs_clsid].incr_hits++;
} else if (op == OP_DECR) {
    c->thread->stats.slab_stats[it->slabs_clsid].decr_hits++;
} else if (op == OP_MULT) {
    c->thread->stats.slab_stats[it->slabs_clsid].mult_hits++;
}
pthread_mutex_unlock(&c->thread->stats.mutex);
```

This means we'll have to modify the `struct slab_stats`.

Well if there are `*_hits`, there are `*_misses` and surely enough when we return from this function back to our `process_arithmetic_command` function we find the logic waiting for us.

```c
case DELTA_ITEM_NOT_FOUND:
    pthread_mutex_lock(&c->thread->stats.mutex);
    if (op == OP_INCR) {
        c->thread->stats.incr_misses++;
    } else if(op == OP_DECR) {
        c->thread->stats.decr_misses++;
    } else if(op == OP_MULT) {
        c->thread->stats.mult_misses++;
    }
    pthread_mutex_unlock(&c->thread->stats.mutex);

    out_string(c, "NOT_FOUND");
    break;
```

We rebuild our code and run it.

```bash
set v 0 360 1        # input
1                    # input
STORED               # output

get v                # input
VALUE v 0 1          # ouput
1                    # output
END                  # output

mult v 10            # input
10                   # output
```

## Credits
- 📖 [The best engineering interview question I’ve ever gotten, Part 1](https://quuxplusone.github.io/blog/2022/01/06/memcached-interview/)
- 👀 [Tsoding's VOD](https://www.youtube.com/watch?v=ey68sKSFJAU)
- 🔨 [nakst's gdb frontend `gf`](https://github.com/nakst/gf)