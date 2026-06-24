# Contribution 2: `textDocument/references` does not treat alias definitions as references of a given symbol

**Contribution Number:** 2  
**Student:** Christian Perez  
**Issue:** [sorbet/sorbet #9447](https://github.com/sorbet/sorbet/issues/9447)  
**Status:** Phase III — Complete

---

## Why I Chose This Issue

I wanted something that actually pushed me into unfamiliar territory. C++ is the first language I learned, so the language itself isn't the barrier — but working inside a compiler's LSP implementation is a different beast than typical application code. That's exactly why I picked it.

The issue is also well-scoped for what it is. It's tagged `good first issue` by a core maintainer, the expected behavior is clearly defined, and there's no ambiguity about what "fixed" looks like. If Find All References on `foo` shows the `alias_method :bar1, :foo` line, it's fixed. That's a clean target.

---

## Understanding the Issue

### Problem Description

Sorbet's LSP server implements `textDocument/references`, which powers the "Find All References" feature in editors. When you search for references of a method like `foo`, it should return every place `foo` is referenced — including its alias definitions. Right now, `alias_method :bar1, :foo` is not included in that list even though renaming `foo` would require updating it.

### Expected Behavior

Running Find All References on `def foo` should include the `alias_method :bar1, :foo` line as a reference, since `:foo` there is a direct reference to the `foo` symbol that would break if `foo` were renamed.

### Current Behavior

Find All References on `def foo` returns:
1. The `def foo` definition itself
2. Direct call sites like `foo` inside other methods

It does **not** return the `alias_method :bar1, :foo` line, even though `:foo` there is a reference to the same symbol.

### Affected Components

- **Language:** C++
- **Files changed:** `main/lsp/DefLocSaver.cc`, `main/lsp/DefLocSaver.h`
- **Test added:** `test/testdata/lsp/alias_method_references.rb`
- **Relevant area:** Sorbet's LSP AST walker that emits query responses for symbol references

---

## Reproduction Process

### Environment Setup

Sorbet uses Bazel as its build system, which was new to me. Here's the full setup on Mac (Apple Silicon):

- **Xcode CLI tools** — already installed (version 2416)
- **Homebrew** — installed fresh
- **Bazelisk** — installed via `brew install bazelisk` (handles Bazel version automatically)
- **Clang** — Apple clang 21.0.0, already present via Xcode

```bash
git clone https://github.com/p2rez/sorbet.git
cd sorbet
git remote add upstream https://github.com/sorbet/sorbet.git
git checkout -b fix/lsp-alias-references
bazel build //main:sorbet  # ~30-60 min first build
```

First build took a while since it compiles the full C++ project from scratch. Subsequent builds were fast (under 10s for single-file changes).

### Steps to Reproduce

1. Open a Ruby file in an editor with Sorbet's LSP running (or use sorbet.run)
2. Create a class with a method and an alias:
   ```ruby
   # typed: true
   class A
     def foo; end
     alias_method :bar1, :foo

     def bar2
       foo
     end
   end
   ```
3. Run Find All References on `def foo`
4. Observe the results

**Result:** Only `def foo` and the `foo` call inside `bar2` are returned. The `alias_method :bar1, :foo` line is missing.

### Reproduction Evidence

- **Branch link:** [p2rez/sorbet — fix/lsp-alias-references](https://github.com/p2rez/sorbet/tree/fix/lsp-alias-references)
- **My findings:** `alias_method` gets desugared to a `Send` node in Sorbet's AST — `self.alias_method(:bar1, :foo)`. The `:foo` argument is an `ast::Literal` with its own source location, but Sorbet's `DefLocSaver` (the AST walker that emits LSP query responses) had no handler for `Send` nodes, so the `:foo` argument was never registered as a symbol reference.

---

## Solution Approach

### Analysis

Sorbet's LSP system works by walking the AST via `DefLocSaver` and emitting `QueryResponse` objects when a node matches the current LSP query (either by location or by symbol). The issue was that `alias_method :bar1, :foo` is desugared to a `Send` node before `DefLocSaver` runs, and there was no `postTransformSend` handler to detect it.

When `textDocument/references` fires for `foo`, the query system never sees the `:foo` argument inside the `alias_method` call — it has no mechanism to emit a response for it.

### Proposed Solution

Add a `postTransformSend` handler in `DefLocSaver` that:
1. Detects `alias_method` calls
2. Looks up the method named by the second argument (`:foo`)
3. If the LSP query matches that method (either by symbol or by cursor location), emits a `MethodDefResponse` at the `:foo` argument's location

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When `textDocument/references` is called for `foo`, Sorbet's `DefLocSaver` walks the AST emitting responses for matching nodes. `alias_method :bar1, :foo` becomes a `Send` node with `:foo` as a symbol literal argument — but no handler existed for this case.

**Match:** `DefLocSaver` already has handlers for `MethodDef`, `UnresolvedIdent`, `ConstantLit`, and `ClassDef`. The pattern is: check if the LSP query matches the node's symbol or location, then push a `QueryResponse`. The new `postTransformSend` follows the same pattern.

**Plan:**
1. Add `postTransformSend` declaration to `DefLocSaver.h`
2. Implement it in `DefLocSaver.cc` — detect `alias_method`, look up the source method, emit `MethodDefResponse` at `:foo` location
3. Add `core/Types.h` include for `core::Types::untyped`
4. Write LSP test at `test/testdata/lsp/alias_method_references.rb`
5. Build and verify with `bazel build //main:sorbet`
6. Run test with `bazel test //test:test_LSPTests/testdata/lsp/alias_method_references`
7. Run full test suite with `bazel test //test:test`

**Implement:** [fix/lsp-alias-references](https://github.com/p2rez/sorbet/tree/fix/lsp-alias-references)

**Review:**
- [x] Follows Sorbet's C++ style conventions
- [x] Existing tests still pass (2243/2243)
- [x] New test covers the alias reference case
- [x] PR description clearly explains what changed and why
- [x] Checked Sorbet's contribution guidelines

**Evaluate:** LSP test passes — Find All References on `foo` now includes the `alias_method :bar1, :foo` line.

---

## Testing Strategy

### Unit Tests

- [x] `textDocument/references` on `foo` returns the `alias_method :bar1, :foo` line — `test/testdata/lsp/alias_method_references.rb`
- [x] `textDocument/references` still returns call sites and the definition (regression) — covered by full test suite
- [x] No regressions on existing references behavior — 2243/2243 tests pass

### Integration Tests

- [x] Full LSP test suite passes with no regressions

### Manual Testing

Verified by running the specific LSP test:
```bash
bazel test //test:test_LSPTests/testdata/lsp/alias_method_references --config=dbg
# PASSED in 2.5s
```

And full suite:
```bash
bazel test //test:test --config=dbg
# Executed 2243 out of 2243 tests: 2243 tests pass.
```

---

## Implementation Notes

### Week 1 Progress

Picked issue #9447 in sorbet/sorbet — the LSP `textDocument/references` handler doesn't include alias definitions as references of a symbol. Tagged `good first issue` by a core maintainer. C++ is my first language so the implementation side should be manageable. Commented on the issue to claim it and set up this journal.

### Week 2 Progress

Got the local environment running. Bazel was new but straightforward once Bazelisk was installed. Built Sorbet successfully (`Sorbet typechecker 0.6.0`). Traced the bug to `DefLocSaver.cc` — the AST walker that emits LSP query responses has no handler for `Send` nodes, meaning `alias_method` calls are completely invisible to the query system.

### Week 3 Progress

Implemented the fix. Key decisions made along the way:

- **Wrong approach first:** Initially tried adding alias symbols to the `symbols` list in `references.cc`. This returned the wrong source location (the whole alias method definition, not the `:foo` argument specifically).
- **Right approach:** Add a `postTransformSend` to `DefLocSaver.cc`. This is where other node types are handled, and it allows emitting a response at the exact `:foo` argument location.
- **`alias_method` desugaring:** Sorbet desugars `alias_method :bar1, :foo` to `self.alias_method(:bar1, :foo)` before `DefLocSaver` runs. The `:foo` arg is an `ast::Literal` with a `loc` field (not `loc()` — learned that from a compiler error).
- **Finding the method:** Used `owner.data(ctx)->findMethodNoDealias(name)` to look up the method by name in the enclosing class without following alias chains.

### Code Changes

- **Files modified:** `main/lsp/DefLocSaver.cc`, `main/lsp/DefLocSaver.h`
- **File added:** `test/testdata/lsp/alias_method_references.rb`
- **Key commit:** [fix/lsp-alias-references](https://github.com/p2rez/sorbet/tree/fix/lsp-alias-references)

---

## Pull Request

**PR Link:** *Phase IV — to be opened*

**PR Description:** *Draft being prepared*

**Maintainer Feedback:**
- *To be updated*

**Status:** Branch pushed, ready to open PR

---

## Learnings & Reflections

### Technical Skills Gained

*To be filled in at the end.*

### Challenges Overcome

*To be filled in at the end.*

### What I'd Do Differently Next Time

*To be filled in at the end.*

---

## Resources Used

- [sorbet/sorbet Issue #9447](https://github.com/sorbet/sorbet/issues/9447)
- [Sorbet LSP source — main/lsp/](https://github.com/sorbet/sorbet/tree/master/main/lsp)
- [Sorbet DefLocSaver source](https://github.com/sorbet/sorbet/blob/master/main/lsp/DefLocSaver.cc)
- [sorbet.run — interactive playground](https://sorbet.run)
- [Bazelisk — Bazel version manager](https://github.com/bazelbuild/bazelisk)
