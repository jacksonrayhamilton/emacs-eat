# Summary for Emacs-Eat Maintainers

## Bug Report

**Issue:** Charset assertion failure causing terminal crashes when processing output from Claude CLI tool

**Error:**
```
Debugger entered--Lisp error: (cl-assertion-failed (charset nil))
  cl--assertion-failed(charset)
  eat--t-write(...)
```

## Root Cause

The charset parser in `eat--t-handle-output` has a false-matching bug when processing incomplete/corrupt charset designation sequences.

**Triggering Sequence:**
```
\e(\r\n     \e[37mcolored
```

**What happens:**
1. `\e(` enters `read-charset-standard` state to read a charset designator
2. Parser buffers subsequent characters: `\r\n     \e[3`
3. Parser searches for any character matching the charset designator regex `[0-9<=>?A-Z...]`
4. Finds `7` in the SGR sequence `\e[37m` at an arbitrary position later in the stream
5. Constructs invalid charset string `"\r\n     \e[37"` which doesn't match any known charset
6. Calls `eat--t-set-charset` with `nil`, corrupting the charset state
7. Later, `eat--t-write` fails assertion when charset lookup returns `nil`

**The Core Problem:** The parser uses `string-match` which searches forward through the entire remaining output, finding charset designator characters in unrelated escape sequences. It doesn't validate that the designator appears immediately after `\e(`.

## The Fix

Modified the `read-charset-standard` case in `eat--t-handle-output` (eat.el:~3847) to:

1. **Check characters one at a time** at the current index, rather than searching forward
2. **Abort immediately** when encountering:
   - Control characters (< 0x20), which indicate a new escape sequence
   - Buffer length exceeding 2 characters (valid charset designators are 1-2 chars max)
3. **Only match designators at the immediate position**, preventing false matches in subsequent sequences
4. **Guard against nil charset values** - only call `eat--t-set-charset` when pcase matches a recognized designator

**Key Change:** Process characters sequentially and validate sequence boundaries, rather than using unbounded forward search.

## Test Coverage

Added comprehensive test `eat-test-character-sets-disambiguation` that verifies:

1. **Bug scenario**: `\e(\e[37m` correctly aborts charset designation and processes the SGR sequence
2. **Color application**: The SGR sequence correctly applies foreground color (not corrupted)
3. **Normal charset switching**: Valid sequences like `\e(0` (DEC line drawing) still work correctly
4. **No regression**: Existing test `eat-test-character-sets` continues to pass

## Files Modified

1. **eat.el** (~line 3847-3883): Fixed charset parser state machine
2. **eat-tests.el** (~line 5732): Added regression test

## Verification

All existing tests pass, including the original `eat-test-character-sets`. The new test fails without the fix and passes with it, confirming the bug is resolved without breaking existing functionality.
