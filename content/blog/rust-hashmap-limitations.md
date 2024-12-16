+++
title = "Rust HashMap limitations"
date = 2024-12-16
updated = 2024-12-16
+++

This post gives examples of API limitations in the Rust standard library's [`HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html). The limitations make some code slower than necessary. The limitations are on the API level. You don't need to change much implementation code to fix them but you need to change stable standard library APIs.

## Entry

HashMap has an [entry API](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.entry). Its purpose is to allow you to operate on a key in the map multiple times while looking up the key only once. Without this API, you would need to look up the key for each operation, which is slow.

Here is an example of an operation without the entry API:

```rust
fn insert_or_increment(key: String, hashmap: &mut HashMap<String, u32>) {
    if let Some(stored_value) = hashmap.get_mut(&key) {
        *stored_value += 1;
    } else {
        hashmap.insert(key, 1);
    }
}
```

This operation looks up the key twice. First in `get_mut`, then in `insert`.

Here is the equivalent code with the entry API:

```rust
fn insert_or_increment(key: String, hashmap: &mut HashMap<String, u32>) {
    hashmap
        .entry(key)
        .and_modify(|value| *value += 1)
        .or_insert(1);
}
```

This operation looks up the key once in `entry`.

Unfortunately, the entry API has a limitation. It takes the key by value. It does this because when you insert a new entry, the hash table needs to take ownership of the key. However, you might not always decide to insert a new entry after seeing the existing entry. In the example above we only insert if there is no existing entry. This matters when you have a reference to the key and turning it into an owned value is expensive.

Consider this modification of the previous example. We now take the key as a string reference rather than a string value:

```rust
fn insert_or_increment(key: &str, hashmap: &mut HashMap<String, u32>) {
    hashmap
        .entry(key.to_owned())
        .and_modify(|value| *value += 1)
        .or_insert(1);
}
```

We had to change `entry(key)` to `entry(key.to_owned())`, cloning the string. This is expensive. It would be better if we only cloned the string in the `or_insert` case. We can accomplish by not using the entry API like in this modification of the first example.


```rust
fn insert_or_increment(key: &str, hashmap: &mut HashMap<String, u32>) {
    if let Some(stored_value) = hashmap.get_mut(key) {
        *stored_value += 1;
    } else {
        hashmap.insert(key.to_owned(), 1);
    }
}
```

But now we cannot get the benefit of the entry API. We have to pick between two inefficiencies.

This problem could be avoided if the entry API supported taking the key by reference (more accurately: by borrow) or by [`Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html). The entry API could then internally use `to_owned` when necessary.

The custom hash table implementation in the hashbrown crate [implements](https://docs.rs/hashbrown/latest/hashbrown/struct.HashMap.html#method.entry_ref) this improvement. [Here](https://internals.rust-lang.org/t/head-desking-on-entry-api-4-0/2156) is a post from 2015 by Gankra that goes into more detail on why the standard library did not do this.

## Borrow

The various HashMap [functions](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.contains_key) that look up keys do not take a reference to the key type. Their signature looks like this:


```rust
pub fn contains_key<Q>(&self, k: &Q) -> bool
where
    K: Borrow<Q>,
    Q: Hash + Eq + ?Sized,
```

They take a type Q, which the hash table's key type can be borrowed as. This happens through the [borrow](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) trait. This makes keys more flexible and allows code to be more efficient. For example, `String` as the key type still allows look up by `&str` in addition of `&String`. This is good because it is expensive to turn `&str` into `&String`. You can only do this by cloning the string. Generic keys through the borrow trait allow us to work with `&str` directly, omitting the clone.

Unfortunately the borrow API has a limitation. It is impossible to implement in some cases.

Consider the following example, which uses a custom key type:

```rust
#[derive(Eq, PartialEq, Hash)]
struct Key {
    a: String,
    b: String,
}

type MyHashMap = HashMap<Key, ()>;

fn contains_key(key: &Key, hashmap: &MyHashMap) -> bool {
    hashmap.contains_key(key)
}
```

Now consider a function that takes two key strings individually by reference, instead of the whole key struct by reference:

```rust
fn contains_key(key_a: &str, key_b: &str, hashmap: &MyHashMap) -> bool {
    todo!()
}
```

How do we implement the function body? We want to avoid expensive clones of the input strings. It seems like this is what the borrow trait is made for. Let's create a wrapper struct that represents a custom key reference. The struct functions `&str` instead of `&String`.

```rust
#[derive(Eq, PartialEq, Hash)]
struct KeyRef<'a> {
    a: &'a str,
    b: &'a str,
}

impl<'a> Borrow<KeyRef<'a>> for Key {
    fn borrow(&self) -> &KeyRef<'a> {
        &KeyRef {
            a: &self.a,
            b: &self.b,
        }
    }
}

fn contains_key(key_a: &str, key_b: &str, hashmap: &MyHashMap) -> bool {
    let key_ref = KeyRef { a: key_a, b: key_b };
    hashmap.contains_key(&key_ref)
}
```

This does not compile. In the borrow function we attempt to return a reference to a local value. This is a lifetime error. The local value would go out of scope when the function returns, making the reference invalid. We cannot fix this. The borrow trait requires returning a reference. We cannot return a value. This is fine for `String` to `&str` or `Vec<u8>` to `&[u8]`, but it does not work for our key type.

This problem could be avoided by changing the borrow trait or introducing a new trait for this purpose.

(In the specific example above, we could workaround this limitation by changing our key type to store `Cow<str>` instead of `String`. This is worse than the `KeyRef` solution because it is slower because now all of our keys are enums.)

The custom hash table implementation in the hashbrown crate implements this improvement. Hashbrown uses a better designed [custom trait](https://docs.rs/hashbrown/0.15.2/hashbrown/trait.Equivalent.html) instead of the standard borrow trait.
