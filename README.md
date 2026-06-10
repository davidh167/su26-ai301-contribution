# Contribution 
#17158: TOCTOU: call `Path::metadata` after `Path::exists`
**Contribution Number:** 1  
**Student:** David Hernandez  
**Issue:** https://github.com/rust-lang/rust-clippy/issues/17158  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I've been teaching myself Rust for about a year. In that time I was able to produce some production-level code, and for this project I primarily looked at issues in tools I was already familiar with. I have some systems programming experience in C from my undergrad work, but I haven't had to touch code this close to the OS in a while.
This issue caught my attention because TOCTOU bugs are something I remember from systems courses. Seeing one surface in a modern, safety-focused language like Rust seemed like an interesting opportunity to me. Although there wasn't much discussion under it and the description was just enough to get an idea of where to start, I noticed it was tagged as a "good first issue," which reassured me that the community was open to new contributors such as myself. I'm hoping to learn how Clippy's lint infrastructure works under the hood and get more comfortable reading and writing code that interacts with the compiler.

---

## Understanding the Issue

### Problem Description

Rust's `Path::exists()` checks whether a file exists by making a syscall to the OS. If code then immediately calls `Path::metadata()` on the same path, that triggers a *second* syscall. Between these two calls, the file system could change — for example, another process could delete or replace the file. This is a classic **Time-Of-Check to Time-Of-Use (TOCTOU)** race condition bug. The issue requests a new Clippy lint that detects this pattern and warns developers about it.

### Expected Behavior

The lint should detect code like:

```rust
if new_path.exists() {
    let md = new_path.metadata().unwrap();
    // ...
}
```

And suggest that it be simplified to:

```rust
if let Ok(md) = new_path.metadata() {
    // ...
}
```

This refactored version only makes **one** syscall, eliminating the race condition entirely.

### Current Behavior

Clippy currently has no lint that catches this pattern. Code that calls `Path::exists()` followed by `Path::metadata()` (or similar path operations) compiles and runs without any warning, even though it introduces a subtle TOCTOU bug.

### Affected Components

- **`clippy_lints/src/`** — where the new lint logic will live (a new file, e.g., `path_toctou.rs` or similar)
- **`clippy_lints/src/lib.rs`** — where the lint module must be registered
- **`tests/ui/`** — where UI test cases (input + expected output) are added for the new lint
- Potentially **`clippy_utils/`** — shared utilities for pattern matching on method calls

---

## Reproduction Process

### Environment Setup

According to Clippy's CONTRIBUTING.md, setting up the dev environment requires:

1. **Install a nightly Rust toolchain** — Clippy is built on nightly Rust. Run `rustup toolchain install nightly` to get it.
2. **Clone the repo** — `git clone https://github.com/rust-lang/rust-clippy && cd rust-clippy`
3. **Build the project** — `cargo build` from the repo root.
4. **Set up editor tooling (recommended)** — For VS Code with rust-analyzer, add `{ "rust-analyzer.rustc.source": "discover" }` to settings so you get proper completions for rustc internals like `Expr` and `EarlyContext`. For RustRover 2026.1+, no extra setup is needed.

### Steps to Reproduce

1. Clone the `rust-lang/rust-clippy` repository and build it locally.
2. Write a small Rust file containing the problematic pattern:
   ```rust
   use std::path::Path;
   fn check(p: &Path) {
       if p.exists() {
           let _md = p.metadata().unwrap();
       }
   }
   ```
3. Run `cargo clippy` on that file — observe that **no warning is emitted**, confirming the lint is missing.

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[To be completed in a future phase.]

### Proposed Solution

[To be completed in a future phase.]

### Implementation Plan

[To be completed in a future phase.]

---

## Testing Strategy

[To be completed in a future phase.]

---

## Implementation Notes

[To be completed in a future phase.]

---

## Pull Request

[To be completed in a future phase.]

---

## Learnings & Reflections

[To be completed in a future phase.]

---

## Resources Used

- [Clippy CONTRIBUTING.md](https://github.com/rust-lang/rust-clippy/blob/master/CONTRIBUTING.md)
- [Clippy development guide](https://doc.rust-lang.org/nightly/clippy/development/index.html)
- [Real-world TOCTOU example in coreutils](https://github.com/uutils/coreutils/pull/11711/changes)
- [Issue #17158](https://github.com/rust-lang/rust-clippy/issues/17158)
- [Rust `std::path::Path` docs](https://doc.rust-lang.org/std/path/struct.Path.html)
