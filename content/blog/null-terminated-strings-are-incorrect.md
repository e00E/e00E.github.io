+++
title = "Null terminated strings are incorrect"
date = 2022-11-25
updated = 2022-11-25
+++

## Introduction

In the C programming language it is common to store text as null terminated strings. A null terminated string is a sequence of bytes ending in a null byte (`0x00`, more than one byte for wide characters) that represents a sequence of characters in some text encoding. Each character is mapped to one or more bytes according to the encoding.

C defines the following string types:
- [byte](https://en.cppreference.com/w/c/string/byte), example ASCII
- [multibyte](https://en.cppreference.com/w/c/string/multibyte), example UTF-8
- [wide](https://en.cppreference.com/w/c/string/wide), example UTF-32

All of them are null terminated. Each comes with syntax for creating [literals](https://en.cppreference.com/w/c/language/string_literal) in source code, and functions that operate on them like [`strcpy`](https://en.cppreference.com/w/c/string/byte/strcpy).

For example, the literal `"hello"` is a null terminated byte string represented in memory as `0x68 0x65 0x6c 0x6c 0x6f 0x00`. The first 5 bytes come from the [ASCII](https://en.cppreference.com/w/c/language/ascii) encoding. The next and last byte is the null byte. It doesn't correspond to any character in the original text. It only indicates the end of the string.

## The problem

Intuitively a programming language is expected to handle any string in the active encoding. It would be surprising if `hello world` was an acceptable string while `hello earth` was not.

C violates this expectation. It does so because it gives special meaning to the null byte. Encodings like ASCII and UTF-8 already assign meaning to the null byte. It encodes the [null character](https://en.wikipedia.org/wiki/Null_character). What exactly the null character means is not relevant. It only matters that it is a valid character and encoded as `0x00`. This meaning clashes with the meaning C gives the null byte. A string supposed to contain the null character is instead cut short at its position.

 For example, `hello\0world` (`\0` is the null character) is a valid ASCII and UTF-8 string that could be a literal in source code or read from a file. It can even be typed on the right kind of keyboard, just like pressing the enter key types the newline `\n` character. But the string functions in the C standard library cannot handle the null character correctly because they treat it as the end of the string like in the following examples:
- `strcpy` only copies `hello`.
- `strcmp` returns that `hello\0world` and `hello\0earth` are equal.
- `printf` only prints `hello`.

This is not a bug in these functions. It is a fault in the C standard for designing the string types with null termination.

## The solution

An alternative way to represent strings is with a pointer and a size. This is done in C++'s `std::string`, `std::string_view` and other languages. This representation does not have C's problem and has other technical benefits unrelated to correctness that this post does not go into.

A historical reason that C chose the null byte representation is that it saves memory. Only one extra byte is needed. This is no longer a good reason and maybe was not one even then. The gain in efficiency is not worth the loss in correctness.

**Use better string types. Do not use C's null terminated string functions. Use libraries that handle strings correctly.**

I would like to link a C library that replicates the standard library's string functions with pointer and size but I do not know one.

You can store string literals containing null bytes in arrays with `const char text[] = "hello\0world";`. This gives you access to the size of the string where a pointer would not. The ending null byte is still added.

## Examples of problematic software

SQLite [claims](https://www.sqlite.org/datatype3.html) its string type stores UTF-8. This is incorrect because SQLite uses null terminated strings. If you try to store a string containing a null byte, it will **silently** be cut off.

Postgres [claims](https://www.postgresql.org/docs/15/multibyte.html) its string type can store UTF-8. This is incorrect because Postgres uses null terminated strings. This is handled better than in SQLite because Postgres [documents](https://www.postgresql.org/docs/15/datatype-character.html) (search for `NUL`) the restriction and errors when a string containing a null byte would be used.
