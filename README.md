# Mockers

Mocking library for Rust.

## Limitations

For now it is not a full-featured mocking library, but just
a prototype to gather feedback. For example, only methods with
two or less arguments are supported, non-'static lifetimes are not
supported and so on.

Mocking magic is implemented using compiler plugin, so **nightly Rust
is required**. It was tested to work with *1.9.0-nightly (924da295c 2016-04-10)*.

## Usage

First of all, you must use *nighly* Rust:

```sh
$ multirust override nighly
```

Add `mockers` and `mockers_macros` as dependencies to your `Cargo.toml`:

```toml
[dependencies]
mockers = "0.2.1"

[dev-dependencies]
mockers_macros = "0.2.1"
```

Say we have `air` crate with some trait and method using this trait:

```rust
// src/lib.rs
#![crate_name = "air"]

pub trait AirConditioner {
    // Note that `make_*` methods receive `&self`
    // and not `&mut self` as they should. This is due
    // to current limitation. It it not inherent and will
    // be lifted in the future.
    fn make_hotter(&self, by: i16);
    fn make_cooler(&self, by: i16);
    fn get_temperature(&self) -> i16;
}

pub fn set_temperature_20(cond: &airconditioner) {
    let t = cond.get_temperature();
    if t < 20 {
        cond.make_hotter(20 + t);
    } else {
        cond.make_cooler(t - 20);
    }
}
```

Import `mockers` crate and `mockers_macros` compiler plugin into test crate:

```rust
// tests/lib.rs

#![feature(plugin)]
#![plugin(mockers_macros)]

extern crate air;
extern crate mockers;
```

Now create mock type for some trait:

```rust
mock!{
    AirConditionerMock,
    air, // This is mocked trait's package.
         // Use `self` to mock local trait.
    trait AirConditioner {
        fn make_hotter(&self, by: i16);
        fn make_cooler(&self, by: i16);
        fn get_temperature(&self) -> i16;
    }
}
```

Note that you have to duplicate trait definition inside `mock!`
macro. This is because compiler plugins work at code AST level
and can't get trait information by it's name.

It is all ready now, lets write test:

```rust
use mockers::Scenario;

#[test]
fn test_make_hotter() {
    let mut scenario = Scenario::new();
    let cond = scenario.create_mock::<AirConditioner>();
    air::set_temperature_20(&cond);
}
```

Run tests:

```
$ cargo test
…
running 1 test
test test_make_hotter ... FAILED

failures:

---- test_make_hotter stdout ----
	thread 'test_make_hotter' panicked at 'Unexpected call of `get_temperature`, no calls are expected', /Users/kriomant/Dropbox/Projects/mockers/src/lib.rs:254
note: Run with `RUST_BACKTRACE=1` for a backtrace.


failures:
    test_make_hotter

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured

       error test failed
```

Seems like call to `get_temperature` was not planned by scenario.
Lets start writing scenario:

```rust
#[test]
fn test_make_hotter() {
    let mut scenario = Scenario::new();
    let cond = scenario.create_mock::<AirConditioner>();

    // Expect that conditioner will be asked for temperature
    // and return 16.
    scenario.expect(cond.get_temperature_call().and_return(16));

    // Expect temperature will be set higher by 4.
    // Event `()` result must be specified explicitly currently.
    scenario.expect(cond.make_hotter_call(4).and_return(()));

    mocked::set_temperature_20(&cond);
}
```

Note that we used `_call` suffix when specifying expected method calls.

Start tests again:

```
…
---- test_make_hotter stdout ----
	thread 'test_make_hotter' panicked at 'called `Result::unwrap()` on an `Err` value: "36 is not equal to 4"', ../src/libcore/result.rs:746
…
```

Oops, seems we have a bug in `set_temperature_20` function. Fix it and test again:

```
…
running 1 test
test test_make_hotter ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
…
```

## Argument matchers

`*_call` mock methods receive argument matchers as parameters. Any value
of required type (which implements `Eq`) can be used as matcher,
in this case it is just compared to actual argument value.

You can use `arg!` macro to match argument values against pattern:

```rust
#[macro_use(arg)]
…
scenario.expect(mock.method_receiving_option_call(arg!(Some(_))).and_return(()));
```

Basic matchers (comparison: `eq`, `ne`, `gt`, …; logical: `not`, `or`, `and`)
are available in `matchers` module.
Specialized matchers (like `is_empty` or `contains`) will be available soon.

You may easily create ad-hoc matchers using `check` method:
```rust
use mockers::matchers::check;
scenario.expect(mock.method(check(|x: &Option<u32>| x.is_some()))
                    .and_return(()));
```

Unfortunately, you can't create polymorphic checker this way (i.e. checker
which may be used for `Option<T>` for any `T`). You have to write own
struct and `MatchArg` implementation in order to do that.

Use `check!` macro instead of `check` function to get slightly more
informative error message:

```rust
#[macro_use(check)]
extern crate mockers;

scenario.expect(mock.method(check!(|x: &Option<u32>| x.is_some()))
                    .and_return(()));
```

## License

Copyright © 2016 Mikhail Trishchenkov

Distributed under the [MIT License](LICENSE).
