+++
title = "Fast float to integer conversions"
date = 2024-11-10
updated = 2024-11-10
+++

In Rust, the standard way of converting floating point values to integers is with the [`as` operator](https://doc.rust-lang.org/reference/expressions/operator-expr.html#type-cast-expressions). This conversion has various guarantees as listed in the reference. One of them is that it saturates: Input values out of range of the output type convert to the minimal/maximal value of the output type.

```rust
assert_eq!(300f32 as u8, 255);
assert_eq!(-5f32 as u8, 0);
```

Saturation comes with a downside. It is slower than not saturating. In C/C++ this kind of cast is [undefined behavior](https://github.com/e00E/cpp-clamp-cast). Rust's casts are never undefined behavior (good), but they are slower than C/C++.

On many [hardware targets](https://doc.rust-lang.org/nightly/rustc/platform-support.html) a float to integer conversion can be done in one instruction. For example [`CVTTSS2SI`](https://www.felixcloutier.com/x86/cvttss2si) on x86_84+SSE. This is the minimum amount of work we need to do to perform the conversion. This is what the C/C++ cast compiles to. Rust needs more instructions because of the additional guarantees.

Sometimes you want faster conversions and don't need saturation. You might already know that your inputs are in range or you might not care about the result for out of range inputs. My new [fast-float-to-integer crate](https://github.com/e00E/fast-float-to-integer) provides this. If in range, then the conversion functions work like the standard `as` operator conversion. If not in range, then you get an unspecified value. We get the speed of the C/C++ version without the undefined behavior.

You never get undefined behavior but you can get unspecified behavior. In the unspecified case, you get an arbitrary value. The function returns and you get a valid value of the output type, but there is no guarantee what that value is. On the assembly level the result is specified but we cannot guarantee this in the crate because the behavior varies between platforms.

## Assembly

We keep track of the generated assembly in the repository and enforce on CI that it is up to date. Here is a typical assembly comparison on x86_64+SSE.

standard:

```asm
f32_to_i64:
    cvttss2si rax, xmm0
    ucomiss xmm0, dword ptr [rip + .L_0]
    movabs rcx, 9223372036854775807
    cmovbe rcx, rax
    xor eax, eax
    ucomiss xmm0, xmm0
    cmovnp rax, rcx
    ret
```

fast:

```asm
f32_to_i64:
    cvttss2si rax, xmm0
    ret
```

As expected, we do the conversion in one instruction. The extra assembly in the standard version performs the saturation.

A more complicated case is f32 to u64. We cannot handle this in one instruction, because the cvttss2si instruction outputs an i64. It cannot handle the values between `i64::MAX` and `u64::MAX`. (AVX512 could do this, but the intrinsics are not stable and it might not be faster.) With some clever optimization, we can still do better than the standard version. I copied the following Rust code from the [repository](https://github.com/e00E/fast-float-to-integer/blob/5ba207a2188031abcf285f8cbd7ef85f7a1f5b8f/src/target_x86_64_sse.rs#L40).

standard:

```asm
f32_to_u64:
    cvttss2si rax, xmm0
    mov rcx, rax
    sar rcx, 63
    movaps xmm1, xmm0
    subss xmm1, dword ptr [rip + .L_0]
    cvttss2si rdx, xmm1
    and rdx, rcx
    or rdx, rax
    xor ecx, ecx
    xorps xmm1, xmm1
    ucomiss xmm0, xmm1
    cmovae rcx, rdx
    ucomiss xmm0, dword ptr [rip + .L_1]
    mov rax, -1
    cmovbe rax, rcx
    ret
```

fast:

```asm
f32_to_u64:
    cvttss2si rcx, xmm0
    addss xmm0, dword ptr [rip + .L_0]
    cvttss2si rdx, xmm0
    mov rax, rcx
    sar rax, 63
    and rax, rdx
    or rax, rcx
    ret
```

First, let us consider a relatively naive branchful solution:

```rust
#[inline(always)]
fn _f32_to_u64_branchful(float: f32) -> u64 {
    const THRESHOLD_FLOAT: f32 = power_of_two_f32(63);
    const THRESHOLD_INTEGER: u64 = 2u64.pow(63);

    let in_range = float <= THRESHOLD_FLOAT;
    if in_range {
        f32_to_i64(float) as u64
    } else {
        // Subtract the threshold from the float. The result is >= 0 because the input is larger
        // than the subtrahend. The result is <= i64::MAX because `u64::MAX - i64::MAX == i64::MAX`.
        let in_range_float = float - THRESHOLD_FLOAT;
        let integer = f32_to_i64(in_range_float) as u64;
        // Overflow is benign because it can only occur for invalid inputs.
        integer.overflowing_add(THRESHOLD_INTEGER).0
    }
}
```

We can turn this into a less naive branchless solution:

```rust
#[inline(always)]
fn f32_to_u64_branchless(float: f32) -> u64 {
    const THRESHOLD: f32 = power_of_two_f32(63);

    let integer1 = f32_to_i64(float);
    let integer2 = f32_to_i64(float - THRESHOLD);
    // If the input is larger than i64::MAX, then integer1 is i64::MIN. This value has 1 as the
    // leftmost bit and 0 as the remaining bits. Right shift on signed values is arithmetic, not
    // logical [1]. We end up with all 0 (in range) or all 1 (out of range).
    let too_large = integer1 >> 63;
    // # If the input is not too large:
    //
    // Integer1 has the correct value. The mask is all 0, which makes the Or result in integer1.
    //
    // # If the input is too large:
    //
    // Integer1 is i64::MIN and the mask is all 1. The Or results in `i64::MIN | integer2`.
    // integer2 has the correct result minus 2**63. This is the correct result without the
    // leftmost bit. The Or adds the missing leftmost bit back.
    (integer1 | (integer2 & too_large)) as u64
}
```

This compiles to the assembly above.
