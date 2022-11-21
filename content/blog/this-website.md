+++
title = "This website"
date = 2022-11-20
updated = 2022-11-21
+++

This post explains how and why this website works.

I want to write simple, plain text documents with little formatting. The natural choice for me is Markdown. I want to avoid dealing with web technologies but still present a rendered version of my texts to web browsers. This can be done with static site generators. The one I am using is [Zola](https://www.getzola.org/).

The source code which generates this site is on [Github](https://github.com/e00E/e00E.github.io). The generated static files are hosted by [Github Pages](https://docs.github.com/en/pages). I use Github Pages with my own domain so that I can migrate whenever I wish. I will likely self host eventually.

I want the generated HTML to be simple and semantic. This means using native elements like `article` instead of `div` when reasonable and giving the browser freedom to present the website how it wants to. For example, I do not enforce a particular color scheme or page width. This allows readers to choose how they want the site to look by using their own CSS or their browser's reader mode.

## Mardown examples

### Code

```rust
fn main() {
    let a = 1;
    let a = 1;
    let a = 1;
    let a = 1;
    let a = 1;
}
```

and `some inline code`.

### List

before list

- list 1
- list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2 list 2
- list 3
    - nested
    - nested
    - nested

after list

1. list 1
2. list 2
3. list 3

### Table

before table

| Col 1 | Col 2 | Col 3 |
| - | - | - |
| item 1 1 | item 1 2 item 1 2 item 1 2 item 1 2 item 1 2 item 1 2 | item 1 3 |
| item 2 1 | item 2 2 | item 2 3 item 2 3 |

after table

### Quote

before quote

> quote 1
>
> quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2. quote 2.
>
> quote 3

after quote
