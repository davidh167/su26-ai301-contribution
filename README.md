# Contribution 
#17158: TOCTOU: call `Path::metadata` after `Path::exists`
**Contribution Number:** 1  
**Student:** David Hernandez  
**Issue:** https://github.com/rust-lang/rust-clippy/issues/17158  
**Status:** Phase III - In Progress

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

- **`clippy_lints/src/`** — where the new lint logic will live (a new file, `path_exists_then_metadata.rs`)
- **`clippy_lints/src/lib.rs`** — where the lint module must be registered
- **`tests/ui/`** — where UI test cases (input + expected output) are added for the new lint
- **`clippy_utils/`** — shared utilities for receiver comparison and type checking

---

## Reproduction Process

### Environment Setup

Followed the steps in Clippy's CONTRIBUTING.md:

1. **Install a nightly Rust toolchain** — `rustup toolchain install nightly`
2. **Clone the repo** — `git clone https://github.com/rust-lang/rust-clippy && cd rust-clippy`
3. **Build the project** — `cargo build` from the repo root
4. **Editor setup** — initially tried VS Code with rust-analyzer, but getting it to play nicely with Rust took long enough that switching to a dedicated IDE felt like the right call. RustRover required no extra configuration and worked out of the box.

### Steps to Reproduce

1. Install a nightly Rust toolchain (`rustup toolchain install nightly`), clone the repo, and run `cargo build` from the root.
2. Create a small Rust file with the pattern:
   ```rust
   use std::path::Path;
   fn check(p: &Path) {
       if p.exists() {
           let _md = p.metadata().unwrap();
       }
   }
   ```
3. Run `cargo clippy` on the file.
4. **Expected:** a warning flagging the TOCTOU pattern.
5. **Actual:** no warning is emitted — confirming the lint doesn't exist yet.

### Reproduction Evidence

- **Working branch:** https://github.com/davidh167/rust-clippy/tree/feat/linter-Path-metadata-after-Path-exists
- **Commit showing reproduction:** [Link to commit in your fork]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

There's no existing lint for this pattern — the fix is to build one. The lint needs to detect an `if` expression whose condition is a call to `.exists()` on a `Path`-like receiver, where the body of that `if` block then calls `.metadata()` on the same receiver. The most structurally similar existing lint is `pathbuf_init_then_push`, which also tracks sequential operations on a path receiver across statements. The key difference is that our lint's primary trigger is an `if` expression, so the entry point is `check_expr` matching `ExprKind::If` rather than a state machine across statements.

### Proposed Solution

A new `LateLintPass` registered as a standalone lint (not inside `methods/`) that:
1. Matches `if <cond> { <then_block> }` where `cond` is a call to `.exists()` on a `Path` or `PathBuf` receiver
2. Checks whether `then_block` calls `.metadata()` on the same receiver using `SpanlessEq` for comparison
3. Emits a warning with a suggestion to replace both calls with `if let Ok(md) = path.metadata()`

### Implementation Plan

**Understand:**

Two variants of the pattern to detect:

```rust
// Variant A — if guard
if path.exists() {
    let md = path.metadata()?;
}

// Variant B — sequential statements
let exists = path.exists();
if exists {
    let md = path.metadata()?;
}
```

Suggested fix for both:

```rust
if let Ok(md) = path.metadata() {
    // use md
}
```

**Match:**

`pathbuf_init_then_push` (`clippy_lints/src/pathbuf_init_then_push.rs`) is the closest structural reference — it detects two sequential operations on a path receiver and is registered as a standalone `LateLintPass`. For type-checking path receivers, `path_buf_push_overwrite.rs` and `path_ends_with_ext.rs` show the `is_diag_item(cx, sym::Path)` pattern to follow.

**Plan:**

1. Scaffold the lint:
   ```bash
   cargo dev new_lint --name=path_exists_then_metadata --pass=late --category=nursery
   ```
   This creates `clippy_lints/src/path_exists_then_metadata.rs`, adds the module entry and registration in `lib.rs`, and stubs out `tests/ui/path_exists_then_metadata.rs`.

2. Declare the lint in the new file using `declare_clippy_lint!` with full documentation (What it does, Why is this bad?, Example, Use instead).

3. Implement `check_expr` on the `LateLintPass` to match `ExprKind::If`, extract the condition's receiver and method name, verify the receiver type is `Path`/`PathBuf`, and check the then-block for a `.metadata()` call on the same receiver via `SpanlessEq::new(cx).eq_expr()`.

4. Implement helper functions:
   - `method_call_name` — extracts receiver + method name from `ExprKind::MethodCall`
   - `is_path_receiver` — type-checks the receiver using `is_diag_item`
   - `block_calls_metadata_on` — walks the then-block for a matching `.metadata()` call
   - `build_suggestion` — constructs the replacement string

5. Register in `lib.rs` — add `mod path_exists_then_metadata` and `store.register_late_pass(...)`.

6. Write UI tests covering positive cases (basic `if path.exists()` + `path.metadata()`, PathBuf variant) and negative cases (exists without metadata, metadata on a different path, metadata outside the if block).

7. Run `TESTNAME=path_exists_then_metadata cargo uibless` to generate `.stderr` and `.fixed` files.

**Implement:** https://github.com/davidh167/rust-clippy/tree/feat/linter-Path-metadata-after-Path-exists

**Review:**
- [ ] Lint naming conventions followed
- [ ] UI tests pass with committed `.stderr` file
- [ ] `cargo test` passes locally
- [ ] `cargo dev update_lints` executed
- [ ] Lint doc block complete (What it does / Why is this bad? / Example / Use instead)
- [ ] `cargo dev fmt` run (requires `rustup component add rustfmt --toolchain=nightly`)
- [ ] `#[clippy::version]` set to current nightly (`rustc -vV` to check)

**Evaluate:**

| Test case | Expected |
|-----------|----------|
| `if path.exists() { path.metadata() }` | lint fires |
| `if path.exists() { path.metadata().unwrap() }` | lint fires |
| PathBuf variant | lint fires |
| `if path.exists() { /* no metadata */ }` | no lint |
| `if path.exists() { other_path.metadata() }` | no lint |
| `if other_path.exists() { path.metadata() }` | no lint |
| Already using `if let Ok(md) = path.metadata()` | no lint |
| `path.metadata()` called after the `if` block | no lint |

---

## Testing Strategy

### UI Tests

Test cases live in `tests/ui/unnecessary_path_exists.rs`. Expected compiler output was generated via `cargo uibless` and committed as `tests/ui/unnecessary_path_exists.stderr`.

**Positive cases (lint fires):** `metadata`, `is_file`, `is_dir`, `is_symlink`, `canonicalize`, `read_dir`, `symlink_metadata`, PathBuf variant, `?` operator, bare statement form, fs op not first in block, deeper method chain.

**Negative cases (no lint):** no fs op in body, else branch present, different receiver, non-fs method, `fs::` free function, compound `&&` condition.

### Manual Testing

Full `cargo test` passed cleanly with zero failures. `cargo dev update_lints` and `cargo dev fmt` were run before the final commit.

### Known Limitations (intentional for v1)

- Does not detect the stored-bool variant: `let exists = path.exists(); if exists { ... }`
- Does not detect `fs::` free functions (e.g. `fs::read(path)`)
- Does not handle `if path.exists() { ... } else { ... }`

---

## Implementation Notes

### Week 3 Progress

**What I built:**

A new Clippy lint — `unnecessary_path_exists` in the `nursery` category — that detects the TOCTOU pattern where `Path::exists()` is used as an `if` condition immediately before a filesystem operation on the same path inside the block body. The lint name changed from the original plan (`path_exists_then_metadata`) after reviewing the issue more carefully. The scope turned out to be broader than just `metadata` — any redundant fs syscall qualifies. Looking at how similar lints are named (`unnecessary_first_then_check`, `unnecessary_get_then_check`) made it clear the convention is to name the unnecessary operation, which led to `unnecessary_path_exists`.

**Challenges faced:**

- `is_diag_item()` is an extension method from `clippy_utils::res::MaybeDef`, which isn't imported by default. The first build failed because of this missing import — the compiler error pointed at it clearly, but it wasn't obvious upfront since other lints pull it in implicitly through their module structure.
- `path.metadata()?` doesn't appear as a simple `MethodCall` in the HIR. The `?` operator desugars to `Match(Call(TryTraitBranch, [inner_expr]), ..., TryDesugar)`, which required an extra match arm in `find_fs_call_in_expr` to peel through the desugaring and reach the inner method call.

**Detected fs methods:** `metadata`, `symlink_metadata`, `canonicalize`, `read_link`, `read_dir`, `is_file`, `is_dir`, `is_symlink`

### Code Changes

- **Files modified:**

| File | Role |
|------|------|
| `clippy_lints/src/unnecessary_path_exists.rs` | Lint implementation |
| `clippy_lints/src/lib.rs` | Module import (autogenerated by `cargo dev update_lints`) |
| `clippy_lints/src/declared_lints.rs` | Lint registration (autogenerated) |
| `tests/ui/unnecessary_path_exists.rs` | UI test cases (positive + negative) |
| `tests/ui/unnecessary_path_exists.stderr` | Expected compiler output (generated by `cargo uibless`) |

- **Key commits:**
  - `c36cead2b` — add unnecessary_path_exists lint to detect TOCTOU after Path::exists
  - `5feeadae5` — expand test coverage for unnecessary_path_exists

- **Still to do before PR:**
  - Run `cargo lintcheck` against real-world crates to check for false positives
  - Document known limitations in the lint's `### Known problems` doc block
  - Squash commits and open PR with changelog entry in the message

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
