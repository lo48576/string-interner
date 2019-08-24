# Unsoundness test

In this branch, changes below are applied.

* Make `InternalStrRef` public (to suppress compiler warning).
* Dump arguments of `InternalStrRef::{partial_eq, hash}()`.
* Make `StringInterner::values` public (to inspect content pointers).
* Add `unsoundness` test.

Run `cargo test unsoundness -- --nocapture` to see what's happening.

```
$ cargo test unsoundness -- --nocapture
   Compiling string-interner v0.7.0 (/home/lo48576/works/public/contrib/string-interner)
    Finished dev [unoptimized + debuginfo] target(s) in 0.37s
     Running target/debug/deps/string_interner-7f1d5487ccb0cecb

running 1 test
InternalStrRef::hash(self=0x55f6d855f22e="foo")
InternalStrRef::hash(self=0x7f3a48000c10="foo")
old = StringInterner {
    map: {
        InternalStrRef(
            0x00007f3a48000c10,
        ): Sym(
            1,
        ),
    },
    values: [
        "foo",
    ],
}
old value addrs = [0x7f3a48000c10]
InternalStrRef::hash(self=0x55f6d855f28a="bar")
InternalStrRef::hash(self=0x7f3a48000d30="bar")
InternalStrRef::hash(self=0x7f3a48000c10="foo")
*** Address to be freed: 0x7f3a48000c10
*** Now the string is freed: 0x7f3a48000c10
new = StringInterner {
    map: {
        InternalStrRef(
            0x00007f3a48000c10,
        ): Sym(
            1,
        ),
    },
    values: [
        "foo",
    ],
}
new value addrs = [0x7f3a48000d10]
new.resolve(foo) = Some("foo")
InternalStrRef::hash(self=0x55f6d855f22e="foo")
InternalStrRef::eq(0x55f6d855f22e="foo", 0x7f3a48000c10="0\u{c}\u{0}")
InternalStrRef::hash(self=0x7f3a48000d30="foo")
InternalStrRef::eq(0x7f3a48000d30="foo", 0x7f3a48000c10="0\u{c}\u{0}")
InternalStrRef::hash(self=0x7f3a48000c10="0\u{c}\u{0}")
new.get_or_intern("foo") = Sym(2)
new = StringInterner {
    map: {
        InternalStrRef(
            0x00007f3a48000c10,
        ): Sym(
            1,
        ),
        InternalStrRef(
            0x00007f3a48000d30,
        ): Sym(
            2,
        ),
    },
    values: [
        "foo",
        "foo",
    ],
}
InternalStrRef::hash(self=0x55f6d855f22e="foo")
InternalStrRef::eq(0x55f6d855f22e="foo", 0x7f3a48000d30="foo")
thread 'tests::unsoundness::inspect_new' panicked at 'assertion failed: `(left == right)`
  left: `Sym(2)`,
 right: `Sym(1)`: `foo` should represent the string "foo" so they should be equal', src/tests.rs:418:3
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
test tests::unsoundness::inspect_new ... FAILED

failures:

failures:
    tests::unsoundness::inspect_new

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 41 filtered out

error: test failed, to rerun pass '--lib'
$
```

In the output above, access to pointer (to destroyed string) `0x7f3a48000c10` is shown,
and its content `"0\u{c}\u{0}"` is broken.
This is because it is already dropped.

This use-after-free is caused by `new.get_or_intern("foo")`.
It fails to compare `"0\u{c}\u{0}"` and `"foo"`, then registers newly created `"foo"`.
This can be seen in second dump of `new`.

```
new = StringInterner {
    map: {
        InternalStrRef(
            0x00007f3a48000c10,  // <- Dangling pointer came from `old`.
        ): Sym(
            1,
        ),
        InternalStrRef(
            0x00007f3a48000d30,  // <- Newly registered `"foo"`.
        ): Sym(
            2,
        ),
    },
    values: [
        "foo",  // <- `"foo": Box<str>` cloned from `old`.
        "foo",  // <- `"foo": Box<str>` which is newly registered by `new.get_or_intern("foo")`.
    ],
}
```
