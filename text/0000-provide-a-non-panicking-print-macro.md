- Feature Name: non_panicking_print
- Start Date: 2015-10-30
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

The println and print macros provides a simple interface to output content on
the stdout of a program. The macros panic when writing to stdout, and the
failure condition could happen on external scenarios. The idea is to implement
a non-panicking pair of print macros.

# Motivation
[motivation]: #motivation

Rust seems ideal tool to write command line applications. Redirecting the
output to pipes is common on such scenarios, but the stdout could get closed on
such scenarios.

Take the following rust code:

```rust
fn main() {
  for _ in 0..10000 {
    println!("line") {
  }
}
```

Piping the program output to other utilities could cause the program to
panic, when the pipe is closed.

```bash
produce_logs | head
line
line
line
line
line
line
line
line
line
line
thread '<main>' panicked at 'failed printing to stdout: Broken pipe (os error 32)', ../src/libstd/io/stdio.rs:588
```

Instead of panicking, it would be interesting to allow the developer to
decide what to do with the error result, either ignoring it or panicking
on it's own. This commit implements `try_println` and `try_print` as an
alternative non-panicking macros.

The following code would not panic anymore when the stdout is closed.

```rust
fn main() {
  for _ in 0..10000 {
    if let Err(_) = try_println!("line") {
      std::process::exit(0);
    }
  }
}
```

One example of when the panicking is not ideal happens on the compiler itself,
as it uses `println!` to output the version, as reported on
rust-lang/rust#14505.

Another discussion on panicking because of stdout has happend on the RFC #1014,
where it was discussed the possibility to let the user deal with errors on a
callback or let it ignore. As all std::io methods return a Result, it would be
interesting to have a sensible default way to print to stdout and let the user
decide what to do in case of failure.

# Detailed design
[design]: #detailed-design

To introduce this pair of macros, little would need to be done.
On src/libstd/io/stdio.rs, the `_print` function would have the result
producing part extracted as another function (eg: `_try_print`), so `_print`
would still panic on the `Err` result, and `_try_print` would return the result
itself.

Then the two macros, `try_print` and `try_println` would be created, in a
similar manner to the others, but delegating to `_try_print`.

An example implementation could be seem on [this commit](https://github.com/bltavares/rust/commit/fb6d310c65f7b077c919e28c82118af559a06399).

# Drawbacks
[drawbacks]: #drawbacks

This would include another way of printing to the standard out as part of the
prelude, which could lead to confusion when to use `println!` or
`try_println!`, as well as increase the size of the prelude macros.

# Alternatives
[alternatives]: #alternatives

Another alternative would be to add on the documentation an example of how to
use `write!` and `std::io::stdout`, like the following code:

```rust
use std::io::Write;

fn main() {
    for _ in 0..10000 {
        if let Err(_) = writeln!(std::io::stdout(), "line") {
            std::process::exit(1);
        }
    }
}
```

The downside of this implementation is that tests currently replace the stdout
of `print!` and `println!`, which will not be possible to do if the code under
test make use of `std::io::stdout`.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
