+++
title = "missed Rust optimizations"
date = 2023-07-14
updated = 2023-07-14
+++

I like inspecting the optimized assembly instructions of small self contained parts of my Rust programs. I sometimes find code that doesn't optimize well. This is especially interesting when it is possible to rewrite the code in a semantically equivalent way which optimizes better. I want to give some examples of this as evidence that it can be worth it to try to improve generated assembly. Compilers are neither perfect nor always better than humans.

The presented examples all come from real code that I wrote. They are not contrived. You can find more examples in the Rust issue tracker with the "slow" [label](https://github.com/rust-lang/rust/issues?q=is%3Aissue+label%3AI-slow).

[cargo-show-asm](https://github.com/pacak/cargo-show-asm) and [Compiler Explorer](https://godbolt.org/) work well for looking at Rust compiler output.

## Example A

[Compiler Explorer](https://godbolt.org/z/4YTjWeWoa), [Rust issue](https://github.com/rust-lang/rust/issues/85841)

We have a simple enum on which we want to implement an iterator style `next` function.



```rust
#[derive(Clone, Copy)]
enum E {
    E0,
    E1,
    E2,
    E3,
}
```

You might implement `next` like this:

```rust
fn next_v0(e: E) -> Option<E> {
    Some(match e {
        E::E0 => E::E1,
        E::E1 => E::E2,
        E::E2 => E::E3,
        E::E3 => return None,
    })
}
```

Which produces this assembly:

```asm
example::next_v0:
        mov     al, 4
        mov     cl, 1
        movzx   edx, dil
        lea     rsi, [rip + .LJTI0_0]
        movsxd  rdx, dword ptr [rsi + 4*rdx]
        add     rdx, rsi
        jmp     rdx
.LBB0_2:
        mov     cl, 2
        jmp     .LBB0_3
.LBB0_1:
        mov     cl, 3
.LBB0_3:
        mov     eax, ecx
.LBB0_4:
        ret
.LJTI0_0:
        .long   .LBB0_3-.LJTI0_0
        .long   .LBB0_2-.LJTI0_0
        .long   .LBB0_1-.LJTI0_0
        .long   .LBB0_4-.LJTI0_0
```

The match expression turns into a jump table with 4 branches. You would expect this assembly, if we did some arbitrary operation in each match case, that isn't related to the other cases. However, If you are familiar with how Rust represents enums and options, you might realize that this is not optimal.

The enum is 1 byte large. The variants are represented as 0, 1, 2, 3. This representation is not guaranteed (unless you use the `repr` attribute) but it is how enums are represented today. The enum only uses 4 out of the 256 possible values. To save space in `Option`s the Rust compiler performs "niche optimization". An option needs one more pattern to represent the empty case `None`. If the inner type has a a free variant, the niche, then it can be used for that. In fact, the representation of `Option::<E>::None` is 4. To implement `next` we just need to increment the byte.

Unfortunately the compiler does not realize this unless we rewrite the function like this:

```rust
fn next_v1(e: E) -> Option<E> {
    match e {
        E::E0 => Some(E::E1),
        E::E1 => Some(E::E2),
        E::E2 => Some(E::E3),
        E::E3 => None,
    }
}
```

Which produces this assembly:

```asm
example::next_v1:
        lea     eax, [rdi + 1]
        ret
```

This is better. There are only two instructions and no branches.

## Example B

[Compiler Explorer](https://godbolt.org/z/6Pv4bxs1b), [Rust issue](https://github.com/rust-lang/rust/issues/113691)

We have an array of 5 boolean values and want to return whether all of them are true.

```rust
pub fn iter_all(a: [bool; 5]) -> bool {
    a.iter().all(|s| *s)
}

pub fn iter_fold(a: [bool; 5]) -> bool {
    a.iter().fold(true, |acc, i| acc & i)
}

pub fn manual_loop(a: [bool; 5]) -> bool {
    let mut b = true;
    for a in a {
        b &= a;
    }
    b
}
```

`iter_all`, `iter_fold`, `manual_loop` produce the same assembly:

```asm
example::iter_all:
        movabs  rax, 1099511627775
        and     rax, rdi
        test    dil, dil
        setne   cl
        test    edi, 65280
        setne   dl
        and     dl, cl
        test    edi, 16711680
        setne   cl
        test    edi, -16777216
        setne   sil
        and     sil, cl
        and     sil, dl
        mov     ecx, 4278190080
        or      rcx, 16777215
        cmp     rax, rcx
        seta    al
        and     al, sil
        ret
```

Usually when several functions have the same assembly they are merged together. This not happening might indicate that the compiler did not understand that all of them do the same thing.

The assembly is an unrolled version of the iterator or loop. Note that the integer constants mask out some bits from a larger pattern like 0xFF00... There is a comparison for every bool. This feels suboptimal because all booleans being true has a single fixed byte pattern that we could compare against together. I try to get the compiler to understand this:

```rust
pub fn comparison(a: [bool; 5]) -> bool {
    a == [true; 5]
}

pub fn and(a: [bool; 5]) -> bool {
    a[0] & a[1] & a[2] & a[3] & a[4]
}
```

```asm
example::comparison:
        movabs  rax, 1099511627775
        and     rax, rdi
        movabs  rcx, 4311810305
        cmp     rax, rcx
        sete    al
        ret

example::and:
        not     rdi
        movabs  rax, 4311810305
        test    rdi, rax
        sete    al
        ret
```

This is better. I'm not sure if `and` is optimal but it is the best version so far.


## Caveats

When I say that some code doesn't optimize well or that some assembly is better, I mean that the code could be compiled into assembly that does the same thing in less time. If you are familiar with assembly, this can be intuited by looking at it. However, the quality of the assembly is not just a product of the instructions. It depends on other things like what CPU you have and what else is going on in the program. You need realistic benchmarks to determine whether some code is faster with high confidence. You might also care less about speed and more about the size of the resulting binary. These nuances do not matter for the examples in this post.

While I was able to rewrite code to improve generated assembly, none of the improvements are guaranteed. With the next compiler version both versions of the code might compile to the same assembly. Or the better version today might become the worse version tomorrow. This is an argument in favor of not worrying too much about the generated assembly and more about other metrics like code clarity. Still, for especially hot loops or especially bad assembly, making these adjustments can be worth it.
