# Vim contribution report

Upstream repository: https://github.com/vim/vim (default branch `master`)
Fork: https://github.com/b84330808/vim
Base commit for all fix branches: `451793f` (upstream `master` on 2026-11-04,
"runtime(doc): add missing comma in gettabwinvar() example", patch level 9.2.1286)

Build: `cd src && make` on Linux (Huge features, no GUI), no compiler warnings
before or after the changes.

## Policy notes (read before opening any PR)

- **AI use is allowed, but must be disclosed** with an `Assisted-by:` trailer
  (CONTRIBUTING.md "Using AI", AGENTS.md, PR template "AI assistance" box).
  The commits on the fix branches carry `Assisted-by: Claude Code`.
- **Commit messages and PR descriptions must be written by you**
  (CONTRIBUTING.md: "Generated commit messages or PR descriptions. Write them
  yourself"). The commit messages on the branches and the PR bodies below are
  only drafts. Rewrite them in your own words before submitting (for example
  with `git commit --amend`). You must also be able to explain every line of
  each change, and answer review questions yourself (pasting AI output there
  is not acceptable).
- No CLA. DCO `Signed-off-by:` is recommended by CONTRIBUTING.md and required by
  AGENTS.md. All commits are made with `git commit -s` as
  `Wei-Lun Li <b84330808@gmail.com>`.
- Commit format: the subject states the problem, followed by `Problem:` and
  `Solution:` lines, with `(Author Name)` at the end of the Solution. Add
  `fixes: #N` for the issue. Maintainers add `closes:` and the `version.c`
  entry when merging, so `src/version.c` is not touched.
- One unrelated issue per PR. A test that fails before the fix and passes after
  it is expected.
- PR template (`.github/pull_request_template.md`) sections: "AI assistance"
  (tick "AI involvement is disclosed in the commit message"), "Checklist"
  (Problem/Solution form, Signed-off-by, Tests, Documentation), and "Anything
  reviewers should know".
- SECURITY.md: send vulnerabilities privately (vim-security@googlegroups.com or
  a GitHub advisory). None of the fixes below is a security issue.
- Text aimed at AI agents: `AGENTS.md` in the repository is guidance for AI
  coding agents (build and test commands, coding style, commit format). It was
  used only as style and format guidance. Issue #21681 ends with
  "Assisted-by: Claude Code" (the reporter's disclosure, not an instruction).
  No repository, issue or PR text tried to redirect this task.

---

## Fix 1: `:defer` cannot call an imported function

- Issue: https://github.com/vim/vim/issues/21700 (open, reported by Galvarcus,
  no linked PR)
- Fix branch: `fix-defer-imported-func`
  (https://github.com/b84330808/vim/tree/fix-defer-imported-func)
- Commit SHA: `527c5c1fe1a813cfa46883b6f8e8f2ff89f85717`
- Files: `src/vim9cmds.c`, `src/vim9execute.c`, `src/testdir/test_vim9_import.vim`

### PR title

    Vim9: :defer cannot call an imported function

### PR body draft (rewrite in your own words)

    Problem:  Vim9: ":defer" cannot call an imported function, it fails with
              E129 (Galvarcus).
    Solution: Also accept a value of type "any" when compiling ":defer" and
              check the type at runtime (Wei-Lun Li).

    An imported function reference "L.Func" is compiled with type "any",
    because for an autoload import the script may not be loaded yet.
    compile_defer() only accepted type "func", so "defer L.Func()" always
    failed, with "import", "import autoload" and a relative import, and also
    when the funcref was first assigned to a variable without a declared type.
    The value is already checked when ":defer" is executed (defer_command()),
    like a funcref call does with PCALL.  The line number is now set there so
    that the error points at the ":defer" line.

    Behavior change: "var x: any = 10 | defer x()" now gives E1029 when
    executed, not E129 when compiling.  A "number" or other non-function type
    still gives E129 at compile time (Test_defer_invalid_func still passes).

    fixes: #21700

    ### AI assistance
    - [x] AI involvement is disclosed in the commit message

    ### Checklist
    - [x] The commit message follows the Problem/Solution form above
    - [x] `Signed-off-by:` trailer is present
    - [x] Tests were added
    - [x] No documentation update is needed (`:help :defer` already documents
          `{func}({args})` without restrictions)

### Root cause

In `compile_load_scriptvar()` (`src/vim9expr.c`) an imported function is
pushed by `generate_PUSHFUNC(cctx, ufunc->uf_name, NULL, TRUE)`, which has
type `any`. For `import autoload` it uses `ISN_LOADEXPORT` / `ISN_AUTOLOAD`,
also with type `any`. `compile_defer()` (`src/vim9cmds.c`) then rejected
anything that is not `VAR_FUNC` with E129. With type `any` it also must not call
`check_func_args_from_type()`, because `t_any` has `tt_argcount == 0`, which
would give "E118: Too many arguments". `defer_command()` already checks for a
func or partial at runtime, and a failed item stays zeroed and is skipped by
`invoke_defer_funcs()`. The legacy `:func` path (`ex_defer`) is not affected:
it was checked and works.

### Test commands

    cd src && make
    cd testdir && make test_vim9_import.res
    # fail-before: revert src/vim9cmds.c and src/vim9execute.c, keep the test
    git stash push src/vim9cmds.c src/vim9execute.c
    cd .. && make && cd testdir && rm -f messages test.log && make test_vim9_import.res
    git stash pop

### Fail-before output (fix reverted, test kept)

    Executed 92 tests                        in   0.534596 seconds
    1 FAILED:

    From test_vim9_import.vim:
    Found errors in Test_defer_imported_func():
    Caught exception in Test_defer_imported_func(): Vim(eval):E129: Function name required @ command line..script src/testdir/runtest.vim[639]..function RunTheTest[63]..Test_defer_imported_func[35]..<SNR>9_CheckScriptSuccess[6]..script src/testdir/XScriptSuccess26[16]..function <SNR>77_Imported, line 1

### Pass-after output

    From test_vim9_import.vim:
    Executed 92 tests                        in   0.483393 seconds
    From test_vim9_script.vim:
    Executed 173 tests                       in   1.845040 seconds
    From test_vim9_disassemble.vim:
    Executed 136 tests                       in   0.280959 seconds
    From test_vim9_func.vim:
    Executed 187 tests                       in   2.292940 seconds
    From test_vim9_class.vim:
    Executed 146 tests                       in   0.415755 seconds

(no `test.log`, so there were no failures)

Valgrind (build with `-DEXITFREE`):

    cd src/testdir
    TEST_FILTER=Test_defer_imported_func make test_vim9_import.res VALGRIND="valgrind --tool=memcheck --leak-check=full --errors-for-leak-kinds=definite --num-callers=45 --log-file=valgrind.defer"

    ==5011==    definitely lost: 0 bytes in 0 blocks
    ==5011== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)

### Competing PR search (all states, 2026-11-04)

`gh api search/issues` with `repo:vim/vim is:pr` and the queries
`defer imported`, `defer import`, `21700`, `"defer" in:title`, `E129`:
none of the results touches this bug. The closest are 18643, 18653 and 18641
(lambda with `:defer`, closed 2025) and 21600 (GC with pending `:defer`,
closed). There is no open or closed PR for #21700, and the issue has no linked PR.

---

## Fix 2: quickfix jump to another help file does not add a jumplist entry

- Issue: https://github.com/vim/vim/issues/21691 (open, reported by
  qwerlkjasdf, no linked PR)
- Fix branch: `fix-qf-help-jumplist`
  (https://github.com/b84330808/vim/tree/fix-qf-help-jumplist)
- Commit SHA: `747409687b26f548a21766bdd3f4d7df6920de74`
- Files: `src/quickfix.c`, `src/testdir/test_quickfix.vim`

### PR title

    quickfix jump to another help file does not add a jumplist entry

### PR body draft (rewrite in your own words)

    Problem:  Jumping to a quickfix or location list entry in another help
              file does not add an entry to the jumplist, CTRL-O does not go
              back to where the cursor was (qwerlkjasdf).
    Solution: Set the previous context mark before editing the help file
              (Wei-Lun Li).

    For a normal file qf_jump_edit_buffer() uses buflist_getfile() with
    GETF_SETMARK, which sets the previous context mark.  For a help file it
    calls do_ecmd() directly, so the mark was never set.  When the buffer
    does not change, qf_jump_to_buffer() already sets the mark, so this is
    only done when going to another buffer.  ":help other.txt" from a help
    window already records the position (via getfile() with setpm), so this
    makes ":cnext"/":cfirst" and the ":l" variants behave the same.

    fixes: #21691

    ### AI assistance
    - [x] AI involvement is disclosed in the commit message

    ### Checklist
    - [x] The commit message follows the Problem/Solution form above
    - [x] `Signed-off-by:` trailer is present
    - [x] Tests were added (quickfix and location list)
    - [x] No documentation update is needed (`:help jump-motions` already
          says "the commands that start editing a new file" are jumps)

### Reproduction of the original report (before the fix)

With `VIMRUNTIME=runtime src/vim --clean -es -S repro.vim`: `:helpgrep \<jumplist\>`,
`:help usr_02.txt`, `:618`, `:cfirst`, `CTRL-O`

    before cfirst: usr_02.txt:618
    after cfirst: builtin.txt:2345
    after CTRL-O: usr_02.txt:1        <- wrong, after the fix: usr_02.txt:618

### Test commands

    cd src && make
    cd testdir && make test_quickfix.res      # or TEST_FILTER=Test_help_jumplist
    # fail-before: revert src/quickfix.c, keep the test
    git stash push src/quickfix.c
    cd .. && make && cd testdir && rm -f messages test.log && make test_quickfix.res
    git stash pop

### Fail-before output (fix reverted, test kept)

    Executed 175 tests                       in   3.438763 seconds
    1 FAILED:

    From test_quickfix.vim:
    Found errors in Test_help_jumplist():
    command line..script src/testdir/runtest.vim[639]..function RunTheTest[63]..Test_help_jumplist[1]..Xhelp_jumplist_tests line 17: Expected ['Xjumpb.txt', 5] but got ['Xjumpa.txt', 3]
    command line..script src/testdir/runtest.vim[639]..function RunTheTest[63]..Test_help_jumplist[2]..Xhelp_jumplist_tests line 17: Expected ['Xjumpb.txt', 5] but got ['Xjumpa.txt', 3]

(both the quickfix `[1]` and the location list `[2]` variants fail)

### Pass-after output

    From test_quickfix.vim:
    Executed 175 tests                       in   3.545496 seconds
    From test_help.vim:
    Executed 16 tests                        in   0.108080 seconds
    From test_help_tagjump.vim:
    Executed 3 tests                         in   0.123233 seconds
    From test_jumplist.vim:
    Executed 4 tests                         in   0.010877 seconds
    From test_tagjump.vim:
    Executed 43 tests                        in   0.270609 seconds

(no `test.log`, so there were no failures). No memory is allocated or freed by this change.

### Competing PR search (all states, 2026-11-04)

`repo:vim/vim is:pr` with `jumplist help`, `jumplist quickfix`, `21691`,
`setpcmark`, `helpgrep jump`, and `"jumplist" in:title created:>2026-01-01`:
nothing related. The closest is 3331 ("curswant is not set on quickfix jumps",
closed 2018), which is a different problem. There is no open or closed PR for #21691.

---

## Fix 3: regexp lookbehind limit `\@N<=` larger than 32 bits is truncated

- Issue: https://github.com/vim/vim/issues/21681 (open, reported by
  ran-extern, no linked PR. The reporter says their PR #21682 for #21668 does
  **not** fix it)
- Fix branch: `fix-regexp-lookbehind-limit`
  (https://github.com/b84330808/vim/tree/fix-regexp-lookbehind-limit)
- Commit SHA: `e10218c784f6e84a688850a05b4e89b9863716ab`
- Files: `src/regexp_bt.c`, `src/regexp_nfa.c`, `src/testdir/test_regexp_latin.vim`
  (the test file is Latin-1 encoded, keep it that way when editing)

### PR title

    regexp: lookbehind limit \@N<= larger than 32 bits is truncated

### PR body draft (rewrite in your own words)

    Problem:  regexp: a limit of 2^32 or more for "\@N<=" and "\@N<!" is
              truncated to 32 bits and acts like a small limit, thus
              "\(ab\)\@4294967297<=c" does not match "abc" (ran-extern).
    Solution: Limit the value to MAXCOL before storing it in the compiled
              pattern (Wei-Lun Li).

    The backtracking engine stores the limit in four bytes (re_put_long())
    and the NFA engine in an int (EMIT()), so 4294967297 became 1.  A line
    cannot be longer than MAXCOL bytes, so a larger limit behaves the same
    as MAXCOL (and as "no limit").  The value from getdecchrs() is checked as
    a vimlong_T before the cast to long, because long is only 32 bits on
    64-bit MS-Windows.  Both "\@N<=" and "\@N<!" are affected and fixed.

    This is independent of #21682 (getdecchrs() wrap-around at 2^64): with
    #21682 values above INT_MAX are still returned (up to INT_MAX * 10 + 9),
    so they still need to be limited here.  The two changes do not conflict;
    a test merge of both passes test_regexp_latin and test_search.

    fixes: #21681
    related: #21668

    ### AI assistance
    - [x] AI involvement is disclosed in the commit message

    ### Checklist
    - [x] The commit message follows the Problem/Solution form above
    - [x] `Signed-off-by:` trailer is present
    - [x] Tests were added (single line, all three 'regexpengine' values, and
          multi-line for both engines)
    - [x] No documentation update is needed

### Test commands

    cd src && make
    cd testdir && make test_regexp_latin.res
    # fail-before: revert the C files, keep the test
    git stash push src/regexp_bt.c src/regexp_nfa.c
    cd .. && make && cd testdir && rm -f messages test.log && make test_regexp_latin.res
    git stash pop
    # the issue's own reproduction
    src/vim --clean -es '+set nocp' "+put =match('abc', '\(ab\)\@4294967297<=c')" "+put =match('abc', '\(ab\)\@99999<=c')" '+2,$print' '+q!'
    #   before: -1 / 2     after: 2 / 2   (same with +set re=1 and re=2)

### Fail-before output (fix reverted, test kept)

    Executed 35 tests                        in   0.616756 seconds
    2 FAILED:

    From test_regexp_latin.vim:
    Found errors in Test_lookbehind_across_line():
    ...Test_lookbehind_across_line line 17: Expected 2 but got 0
    ...Test_lookbehind_across_line line 18: Expected 2 but got 0
    ...Test_lookbehind_across_line line 19: Expected 0 but got 2
    ...Test_lookbehind_across_line line 20: Expected 0 but got 2
    Found errors in Test_regexp_single_line_pat():
    ...line 481: Error 0: pat: \"\(ab\)\@4294967297<=c\", text: \"abc\", did not match, expected: \"c\"
    ...line 481: Error 1: pat: \"\(ab\)\@4294967297<=c\", text: \"abc\", did not match, expected: \"c\"
    ...line 481: Error 2: pat: \"\(ab\)\@4294967297<=c\", text: \"abc\", did not match, expected: \"c\"
    ...line 485: Error 0: pat: \"\(ab\)\@4294967297<!c\", text: \"abc\", match: \"c\", expected no match
    ...line 485: Error 1: pat: \"\(ab\)\@4294967297<!c\", text: \"abc\", match: \"c\", expected no match
    ...line 485: Error 2: pat: \"\(ab\)\@4294967297<!c\", text: \"abc\", match: \"c\", expected no match

(`...` is `command line..script src/testdir/runtest.vim[639]..function RunTheTest[63]..`)

### Pass-after output

    From test_regexp_latin.vim:
    Executed 35 tests                        in   0.644519 seconds
    From test_regexp_utf8.vim:
    Executed 29 tests                        in   0.596188 seconds
    From test_search.vim:
    Executed 85 tests                        in   8.637470 seconds
    From test_substitute.vim:
    Executed 62 tests                        in   0.191833 seconds
    From test_regex_char_classes.vim:
    Executed 3 tests                         in   0.013356 seconds

(no `test.log`, so there were no failures)

### Competing PR search (all states, 2026-11-04)

`repo:vim/vim is:pr` with `lookbehind limit`, `look-behind`, `21681`,
`regexp truncated`, `getdecchrs`, `regexp 32 bits`: the only related PR is
https://github.com/vim/vim/pull/21682 (open, ran-extern, "very large decimal
numbers in \%d or [\d] wrap around in regexp", `fixes: #21668`). It only
changes `getdecchrs()` in `src/regexp.c`, and its author says in #21681 that
it does not fix this. There's no overlap with this branch. Consider mentioning
#21682 in the PR and offering to coordinate with ran-extern. Nothing else
related.

---

## Fix 4: the old regexp engine truncates a large number in `\%l`, `\%c` and `\%v`

- Issue: none upstream. Found by reading the code while working on fix 3
  (same root cause: a number from the pattern stored in 32 bits). Searched
  `repo:vim/vim is:issue` for `E951`, `regexp value too large`,
  `regexp 4294967297` and `regexp truncated 32`: only #21668 and #21681 are
  related, and neither covers `\%l`/`\%c`/`\%v`.
- Fix branch: `fix-regexp-bt-percent-too-large`
  (https://github.com/b84330808/vim/tree/fix-regexp-bt-percent-too-large)
- Commit SHA: `44f2ca296296f3c7db4d59e94ae64751cd6e0c38`
- Files: `src/regexp_bt.c`, `src/testdir/test_regexp_latin.vim` (Latin-1)
- Independent of fix 3. Both change `regexp_bt.c` (fix 3 `regpiece()`, fix 4
  `regatom()`) and `test_regexp_latin.vim` in different places, and a test
  merge of the two branches has no conflicts.

### PR title

    regexp: old engine truncates a large number in \%l, \%c and \%v

### PR body draft (rewrite in your own words)

    Problem:  regexp: the backtracking engine silently truncates a large number
              in "\%23l", "\%23c" and "\%23v" to 32 bits, thus
              "\%#=1\%4294967298c" matches in column 2, while the NFA engine
              gives E951.
    Solution: Check the number in the backtracking engine with the same limits
              as the NFA engine and give E951, avoid wrapping around when
              parsing the digits (Wei-Lun Li).

    The backtracking engine stores the number with re_put_long(), which only
    keeps 32 bits, and the digits were added up without an overflow check
    (a 20-digit number wrapped around at 2^64).  The NFA engine gives E951
    for a value >= INT_MAX (>= INT_MAX / MB_MAXBYTES for "\%v").  The same
    limits are now used for the backtracking engine.  When parsing, the
    value saturates at INT_MAX once it is larger than INT_MAX / 10, which
    is also safe when long_u is 32 bits.  With the default 'regexpengine'
    the NFA engine already rejected these values, so only "\%#=1" or
    'regexpengine' set to 1 was affected.

    Examples with "set re=1" before / after:
      match('abc', '\%4294967298c')   1 / E951
      search('\%4294967298l')         line 2 / E951
      match('abc', '\%2147483647c')   -1 / E951   (NFA: E951)

    ### AI assistance
    - [x] AI involvement is disclosed in the commit message

    ### Checklist
    - [x] The commit message follows the Problem/Solution form above
    - [x] `Signed-off-by:` trailer is present
    - [x] Tests were added
    - [x] No documentation update is needed (E951 is documented under
          `/\%l` without mentioning an engine)

### Test commands

    cd src && make
    cd testdir && make test_regexp_latin.res
    # fail-before: revert src/regexp_bt.c, keep the test
    git stash push src/regexp_bt.c
    cd .. && make && cd testdir && rm -f messages test.log && make test_regexp_latin.res
    git stash pop

### Fail-before output (fix reverted, test kept)

    Executed 35 tests                        in   1.461643 seconds
    1 FAILED:

    From test_regexp_latin.vim:
    Found errors in Test_regexp_error():
    ...Test_regexp_error line 18: command did not fail: call matchstr('abcd', '\%#=1\%2147483647c')
    ...Test_regexp_error line 19: command did not fail: call matchstr('abcd', '\%#=1\%2147483647l')
    ...Test_regexp_error line 20: command did not fail: call matchstr('abcd', '\%#=1\%71582788v')
    ...Test_regexp_error line 21: command did not fail: call matchstr('abcd', '\%#=1\%4294967298c')
    ...Test_regexp_error line 22: command did not fail: call matchstr('abcd', '\%#=1\%<4294967298l')
    ...Test_regexp_error line 23: command did not fail: call matchstr('abcd', '\%#=1\%18446744073709551618v')

(`...` is `command line..script src/testdir/runtest.vim[639]..function RunTheTest[63]..`)

### Pass-after output

    From test_regexp_latin.vim:
    Executed 35 tests                        in   1.978630 seconds
    From test_regexp_utf8.vim:
    Executed 29 tests                        in   0.643245 seconds
    From test_search.vim:
    Executed 85 tests                        in   9.629118 seconds
    From test_substitute.vim:
    Executed 62 tests                        in   0.177858 seconds
    From test_regex_char_classes.vim:
    Executed 3 tests                         in   0.012951 seconds
    From test_syntax.vim:
    Executed 32 tests                        in   0.552465 seconds
    From test_matchadd_conceal.vim:
    Executed 12 tests                        in   0.259043 seconds

(no `test.log`, so there were no failures)

Valgrind (`-DEXITFREE` build, the new error path):

    TEST_FILTER=Test_regexp_error make test_regexp_latin.res VALGRIND="valgrind --tool=memcheck --leak-check=full --errors-for-leak-kinds=definite --num-callers=45 --log-file=valgrind.re"
    ==844==    definitely lost: 0 bytes in 0 blocks
    ==844== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)

### Competing PR search (all states, 2026-11-04)

`repo:vim/vim is:pr` with `E951`, `"value too large"`,
`regexp column number large`, `regexp_bt re_put_long` and
`backtracking engine number`: nothing related. The closest are 9972 (NFA E1204,
closed 2022) and 16994 (`\U`/`\%U` limit, closed 2025). No PR for this.

---

## Other findings in the same module (not fixed, possible follow-ups)

1. Backtracking engine `\{n,m}`: `match('aaa', 'a\{4294967297}')` returns 0
   with `re=1`, because the count is truncated to 32 bits by
   `reginsert_limits()`. Same root cause as fixes 3 and 4. No upstream issue
   yet. Deciding between an error and a clamp here needs more thought
   (`MAX_LIMIT` is `32767 << 16`).

---

## Summary of fix branches

| # | Branch | SHA | Upstream issue |
|---|--------|-----|----------------|
| 1 | `fix-defer-imported-func` | `527c5c1fe1a813cfa46883b6f8e8f2ff89f85717` | #21700 |
| 2 | `fix-qf-help-jumplist` | `747409687b26f548a21766bdd3f4d7df6920de74` | #21691 |
| 3 | `fix-regexp-lookbehind-limit` | `e10218c784f6e84a688850a05b4e89b9863716ab` | #21681 |
| 4 | `fix-regexp-bt-percent-too-large` | `44f2ca296296f3c7db4d59e94ae64751cd6e0c38` | none (found in code) |

All four branch from upstream `master` at `451793f`, each has a single commit
authored and signed off by `Wei-Lun Li <b84330808@gmail.com>`, and each
builds with `make` on Linux with no warnings.
