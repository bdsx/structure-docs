# Basic Hooks

[BDSX-core](https://github.com/bdsx/bdsx-core) hooks BDS to make JS usable in BDS.

The **pseudo** code for original:

```rust
fn START_BDS();

fn main() {
    // ...
    START_BDS();
    // ...
}
```

BDSX-core changes it like below:

```rust
fn START_BDS();
fn START_NODE();
fn START_BDSX_SCRIPTS();

fn main() {
    // ...
    START_NODE();
    // ...
}

fn START_NODE() {
    // ...
    START_BDSX_SCRIPTS();
    // ...
}

fn START_BDSX_SCRITPS() {
    // ...
    START_BDS();
    // ...
}
```
Note: above codes are just pseudo codes. actually they're implemented C++, and JS.
