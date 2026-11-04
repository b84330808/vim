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
