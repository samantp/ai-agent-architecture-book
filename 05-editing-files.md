# Chapter 5 — Editing Files

The edit tool is where coding agents most often fail, and where the best ones invest their most careful engineering. The problem: the model must specify a change to a file it holds only an *imperfect memory* of — it read the file thousands of tokens ago, tokenization mangles whitespace, other edits may have landed since. This chapter examines the three independent, production-hardened solutions in this workspace. They disagree on format but converge — precisely — on the same algorithmic idea.

## 5.1 The design space

| Format | Model emits | Failure mode | Used by |
|---|---|---|---|
| Whole-file rewrite | complete new content | silent truncation/paraphrase of untouched code; O(file) tokens | fallback everywhere (`write`) |
| Search/replace | `oldString` + `newString` | `oldString` doesn't match; matches twice | opencode `edit`, SWE-agent `str_replace_editor`, Cline |
| Custom patch DSL | multi-file, hunk-based patch | hunk context doesn't match | Codex `apply_patch` |
| Line-number edits | range + new content | stale line numbers after any prior edit | (largely abandoned) |

Line-number formats died because agents edit iteratively and every edit invalidates subsequent numbers. Whole-file rewrite survives only as an explicit fallback ("use `write` for an intentional full-file replacement"). The real contest is search/replace vs. patch DSL, and the interesting machinery is the same in both: **fuzzy matching with strictly ordered fallbacks, wrapped in fail-closed guards**.

## 5.2 opencode: the replacer chain

`opencode@34e5809/packages/opencode/src/tool/edit.ts` **[Verified]**. A `Replacer` is a generator producing *candidate strings* that might correspond to the model's `oldString` in the actual file:

```typescript
// edit.ts:217
export type Replacer = (content: string, find: string) => Generator<string, void, unknown>
```

Nine replacers run in strictness order (`edit.ts:694-704`): `SimpleReplacer` (exact), `LineTrimmedReplacer`, `BlockAnchorReplacer`, `WhitespaceNormalizedReplacer`, `IndentationFlexibleReplacer`, `EscapeNormalizedReplacer`, `TrimmedBoundaryReplacer`, `ContextAwareReplacer`, `MultiOccurrenceReplacer`. Each targets a *specific, named model failure*:

- `LineTrimmedReplacer` (`:248`) — per-line comparison after `trim()`: forgives wrong indentation and trailing whitespace, then reconstructs exact byte offsets to yield the *real* text.
- `EscapeNormalizedReplacer` (`:499`) — unescapes `\n`, `\t`, `\"`… : forgives JSON-escaping leakage from the tool-call payload.
- `BlockAnchorReplacer` (`:288`) — the deep cut. When the model gets a block's *first and last lines* right but flubs the middle:

```typescript
// edit.ts:300-356 (abridged)
const firstLineSearch = searchLines[0].trim()
const lastLineSearch  = searchLines[searchLines.length - 1].trim()
const maxLineDelta = Math.max(1, Math.floor(searchBlockSize * 0.25))
// candidates: positions where first+last lines match and the block size
// is within ±25% of what the model claimed
...
// single candidate → accept if middle-line Levenshtein similarity clears
// SINGLE_CANDIDATE_SIMILARITY_THRESHOLD; multiple candidates → stricter
const distance = levenshtein(originalLine, searchLine)
similarity += (1 - distance / maxLen) / linesToCheck
```

Anchors + size sanity check + edit-distance similarity — a genuine approximate-matching algorithm hiding in a "simple" edit tool.

The driver enforces the safety rules — `edit.ts:682-737` **[Verified]**:

```typescript
for (const replacer of [SimpleReplacer, LineTrimmedReplacer, ...]) {
  for (const search of replacer(content, oldString)) {
    const index = content.indexOf(search)
    if (index === -1) continue
    if (isDisproportionateMatch(search, oldString)) throw new Error(
      "Refusing replacement because the matched span is much larger than oldString. Re-read the file...")
    if (replaceAll) return content.replaceAll(search, newString)
    if (index !== content.lastIndexOf(search)) continue   // ambiguous → keep looking
    return content.slice(0, index) + newString + content.slice(index + search.length)
  }
}
throw notFound
  ? Error("Could not find oldString in the file. It must match exactly, including whitespace...")
  : Error("Found multiple matches for oldString. Provide more surrounding context to make the match unique.")
```

Four load-bearing properties:

1. **Ordered leniency** — the exact match always wins if it exists; fuzzier interpretations only get a chance when stricter ones fail.
2. **Uniqueness or nothing** — an ambiguous match is *skipped*, not guessed at. Wrong-location edits are worse than failed edits, because the model won't notice.
3. **Proportionality guard** — `isDisproportionateMatch` rejects a candidate span vastly larger than what the model asked to replace (≥ `old+3` lines or `2×` lines, or 4×/+500 chars — `edit.ts:731-737`): a fuzzy matcher must not be allowed to "match" half the file.
4. **Errors teach.** Each failure message tells the model the corrective action (re-read the file; add surrounding context). The error strings are part of the prompt engineering (Ch. 3.5).

## 5.3 Codex: a patch DSL with the same matcher inside

Codex rejects search/replace in favor of a multi-file patch envelope — taught to the model as a grammar in the system prompt (`open-interpreter@764a96e/codex-rs/core/prompt_with_apply_patch_instructions.md:279+` **[Verified]**):

```
Patch      := "*** Begin Patch" NL { FileOp } "*** End Patch" NL
FileOp     := AddFile | DeleteFile | UpdateFile
AddFile    := "*** Add File: " path NL { "+" line NL }
DeleteFile := "*** Delete File: " path NL
UpdateFile := "*** Update File: " path NL [ "*** Move to: " newPath NL ] { Hunk }
Hunk       := "@@" [ header ] NL { (" "|"-"|"+") text NL }
```

Context is 3 lines before/after each change; when 3 lines can't uniquely locate a hunk, the model escalates to `@@ class BaseClass` / stacked `@@` headers — **scope-qualified addressing instead of more context lines**. Compared to search/replace: one call edits many files atomically, renames are first-class, and add/delete are explicit ops rather than degenerate replacements.

But when a hunk's context must be located in the file, the problem is identical to opencode's — and so is the solution. `open-interpreter@764a96e/codex-rs/apply-patch/src/seek_sequence.rs` **[Verified]**:

```rust
/// Matches are attempted with decreasing strictness: exact match, then
/// ignoring trailing whitespace, then ignoring leading and trailing whitespace.
/// When `eof` is true, we first try starting at the end-of-file...
pub(crate) fn seek_sequence(lines: &[String], pattern: &[String], start: usize, eof: bool)
    -> Option<usize>
{
    // pass 1: exact slice equality
    // pass 2: lines[i+j].trim_end() == pat.trim_end()
    // pass 3: lines[i+j].trim()     == pat.trim()
}
```

Same ordered-leniency idea, in Rust, written independently. Additional details that reward study: the `eof` flag anchors end-of-file hunks by trying `lines.len() - pattern.len()` *first* (patterns meant for the file's end should bind there, not at an earlier coincidental match); the guard for `pattern.len() > lines.len()` carries a comment dating the panic it fixed — patch appliers accumulate scar tissue.

The tool contract is fail-closed and *verified*: `ApplyPatchAction`/`AppliedPatchDelta` track applied changes and whether the application `is_exact()` (`open-interpreter@764a96e/codex-rs/apply-patch/src/lib.rs:184-211` **[Verified]**), and the system prompt then leans on the contract — *"Do not waste tokens by re-reading files after `apply_patch`. The tool call will fail if it didn't work."* The prompt promise and the runtime guarantee are one design.

## 5.4 SWE-agent: the ACI argument, and linting-gated edits

SWE-agent's `str_replace_editor` (`SWE-agent@1132b3e/tools/edit_anthropic/bin/str_replace_editor`, a Python program installed *into the sandbox* — the ACI/"Agent-Computer Interface" thesis: don't give the model human tools, design tools for models) enforces the same rules at its stricter extreme — no fuzzy fallbacks at all **[Verified]**:

```
# :523-537 (paraphrased from source)
occurrences = content.count(old_str)
0 →  "No replacement was performed, old_str `{}` did not appear verbatim in {}."
>1 → "No replacement was performed. Multiple occurrences of old_str `{}` in lines {}.
      Please ensure it is unique"
old == new → "No replacement was performed, old_str is the same as new_str."
```

For a benchmark agent, determinism beats forgiveness: a fuzzy match that guesses wrong poisons the trajectory, while a hard failure just costs one requery. The repo also preserves the evolutionary record — `SWE-agent@1132b3e/tools/windowed_edit_replace`, `windowed_edit_linting`, `windowed_edit_rewrite` **[Verified structure]**: the earlier "windowed" file interface (view N lines, edit within the window) with a variant that ran a **linter as an edit gate**, rejecting edits that introduce syntax errors — verification pulled forward into the mutation itself, the same move as shellcheck-before-execution (Ch. 2.2).

## 5.5 Convergent design rules

Three teams, three languages, one algorithm. The rules they converged on:

1. **Exact first, fuzz in strictly decreasing strictness, stop at first success.**
2. **Never act on an ambiguous match.** Uniqueness is a hard precondition (opencode skips; SWE-agent errors with the line numbers; Codex demands more context or `@@` scoping).
3. **Bound the fuzz.** Proportionality guards (opencode), block-size deltas ±25%, similarity thresholds — leniency without limits is corruption.
4. **Fail closed, and advertise the guarantee to the model** so it doesn't burn tokens re-verifying.
5. **Error messages are corrective prompts** with a specific next action.
6. **Whole-file `write` remains available but separate** — an intentional act, never a fallback the edit tool silently degrades into.
7. Validate *after* applying: track whether application was exact (Codex `is_exact`), diff the result (Cline hosts surface a diff view; opencode computes file diffs per edit for the UI), and feed LSP diagnostics back (Ch. 7.4).

If you build one algorithm from this book, build the replacer chain with uniqueness and proportionality guards — it is ~300 lines and it is the difference between an agent that edits reliably and one that destroys files.
