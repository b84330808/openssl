# OpenSSL contribution report (cloud session)

- Upstream repository: https://github.com/openssl/openssl (default branch `master`)
- Upstream base for every branch below: `ab0917941a00a46a25949c182a7c2df171d96e15`
  ("Dependabot update: bump cross-platform-actions/action", 2026-11-04)
- Fork: https://github.com/b84330808/openssl
- Commit author on every fix: `b84330808 <b84330808@gmail.com>` (OpenSSL uses a
  CLA, not DCO, so there is no `Signed-off-by:`)

## Before you open any PR (read this first)

1. **CLA.** Each fix is non-trivial, so `CLA: trivial` does not apply. The AI
   policy also requires the **new CLA with the AI clauses** (v1.1 or later,
   signed after June 2026). A CLA signed earlier is **not** enough for
   AI-assisted work. Sign it before opening a PR:
   https://openssl-library.org/policies/cla/
2. **Build and test it yourself.** CONTRIBUTING.md: "Before opening a pull
   request you must have read the change in full, understood why it is
   correct, built and tested it yourself, and satisfied yourself that the
   problem it fixes is real." Use the WSL steps below.
3. **Complete the `Assisted-by:` trailer.** Each commit ends with
   `Assisted-by: Claude Code`. The policy format is `Assisted-by: {agent}:{model}`
   and it should name the model. This cloud session was not allowed to write a
   model identifier into repository content. Take the model shown for the
   session that produced the work and amend, for example:
   `git commit --amend` and change the line to `Assisted-by: Claude Code:<model-id>`.
4. **Rate limit.** CONTRIBUTING.md asks for no more than 3-4 open PRs at a time,
   no bulk submission, and "send the ones you consider most important ...
   and explain in each why it matters". Open them one at a time (ranking below).

## Fixes

| # | Branch | Head SHA | Upstream issue | Status |
|---|--------|----------|----------------|--------|
| 1 | `fix/lhash-doall-null-thunk` | `9ebdf9727e367b23e333ce462fad6cded3e12242` | https://github.com/openssl/openssl/issues/33496 | fail-before / pass-after verified |

---

### Fix 1: `OPENSSL_LH_doall()` calls a NULL thunk on `OPENSSL_LH_new()` tables

- Branch: `fix/lhash-doall-null-thunk`
- SHA: `9ebdf9727e367b23e333ce462fad6cded3e12242`
- Files: `crypto/lhash/lhash.c`, `test/lhash_test.c`
- Kind: functional crash (NULL function-pointer call) through public API.
  Not reachable from untrusted input; the table type is chosen by the
  application, so this is not a security issue.

**PR title:** `lhash: fix doall on tables created by OPENSSL_LH_new()`

**PR body draft:**

```
OPENSSL_LH_doall() and OPENSSL_LH_doall_arg() always invoke the per-table
doall thunks (lh->daw / lh->daaw). Those are only set by
OPENSSL_LH_set_thunks(), which only lh_TYPE_new() calls. A table created with
OPENSSL_LH_new() directly has them NULL, so iterating any non-empty table of
that kind calls a NULL function pointer and the caller's callback never runs.

getrn() already falls back to the plain hash/compare functions when their
thunks are NULL. This change does the same for the doall callbacks in
doall_util_fn().

The documentation describes OPENSSL_LH_new()/OPENSSL_LH_doall()/
OPENSSL_LH_doall_arg() as the non-type-specific equivalents of lh_TYPE_*().
TXT_DB_create_index() also builds the db->index[] tables of the public TXT_DB
struct with OPENSSL_LH_new(), so lh_OPENSSL_STRING_doall() on those crashed
as well.

A new test (test_raw_lhash_doall in test/lhash_test.c) iterates an
OPENSSL_LH_new() table with both functions. Without the fix it crashes at
lhash.c:220; with the fix it passes.

Fixes: #33496

##### Checklist
- [x] tests are added or updated
```

**Test commands (run from a build directory, see WSL steps):**

```
make test TESTS=test_lhash V=1
```

**Fail-before** (fix reverted, test kept; ASan+UBSan build):

```
#     ok 1 - test_int_lhash
==29576==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x000000000000 ...)
    #0 0x0  (<unknown module>)
    #1 0x55991066ef6a in OPENSSL_LH_doall ../openssl/crypto/lhash/lhash.c:220
    #2 0x559910536805 in test_raw_lhash_doall ../openssl/test/lhash_test.c:230
    #3 0x5599105434a5 in run_tests ../openssl/test/testutil/driver.c:493
    #4 0x559910546685 in main ../openssl/test/testutil/main.c:145
SUMMARY: AddressSanitizer: SEGV (<unknown module>)
../../util/wrap.pl ../../test/lhash_test => 1
not ok 1 - running lhash_test
Result: FAIL
```

**Pass-after:**

```
#     ok 1 - test_int_lhash
#     ok 2 - test_raw_lhash_doall
../../util/wrap.pl ../../test/lhash_test => 0
ok 1 - running lhash_test
Result: PASS
```

**Competing PR search** (GitHub search over openssl/openssl, all states,
2026-11-04): `is:pr 33496` returned nothing. Keyword searches `OPENSSL_LH_doall`,
`doall_util_fn` and `lhash thunk` (PRs created since 2026-05-01) found no
related PR. The only hit for `lhash thunk` was #32758, a CHANGES/NEWS update
that does not touch this.

---

## Build and test in WSL Ubuntu

```
sudo apt-get update && sudo apt-get install -y build-essential perl git
git clone https://github.com/b84330808/openssl.git && cd openssl
git remote add upstream https://github.com/openssl/openssl.git
git fetch upstream master
git checkout fix/lhash-doall-null-thunk        # or another fix branch
./Configure -d --strict-warnings
make -j"$(nproc)"
make test TESTS=test_lhash V=1                 # per-fix test, see each section
```

Optional sanitizer build (the one used for the evidence above), in a separate
directory:

```
mkdir ../build-asan && cd ../build-asan
../openssl/Configure -d enable-asan enable-ubsan --strict-warnings
make -j"$(nproc)" && make test TESTS=test_lhash V=1
```

To reproduce fail-before for a fix, keep the test and revert only the library
change. For fix 1:

```
git checkout upstream/master -- crypto/lhash/lhash.c
make -j"$(nproc)" && make test TESTS=test_lhash V=1   # expect FAIL
git checkout HEAD -- crypto/lhash/lhash.c
```

Full regression run before opening a PR: `make test` (or `make test HARNESS_JOBS=$(nproc)`).

Formatting: CI uses clang-format 21.1.6 (`.pre-commit-config.yaml`). Check with
`pip install clang-format==21.1.6`, then
`clang-format --style=file <file> | diff -u <file> -`.
All changed files on these branches pass this check.

## Policy notes (read from upstream on 2026-11-04)

- **AI policy** (`openssl/general-policies`, `policies/ai-policy.md`; the
  openssl-library.org site could not be reached from the cloud session):
  AI-assisted contributions are **allowed**. Non-trivial AI-generated content
  needs both (1) an `Assisted-by: {agent}:{model}` trailer in every such commit
  and (2) the new CLA with AI clauses. A human must direct the work, review it,
  test it and be accountable for it.
- **CONTRIBUTING.md**: one logical change per PR; at most 3-4 open PRs; no bulk
  submission; commit title 50-70 chars; self-contained commit messages;
  `Fixes: <issue URL>` line; tests required where possible; `--strict-warnings`
  clean; clang-format (WebKit style); CHANGES.md only for user-visible changes;
  no merge commits; fixup commits during review.
- **PR template** (`.github/PULL_REQUEST_TEMPLATE.md`): description, then a
  checklist "documentation is added or updated" / "tests are added or updated"
  (remove the items that don't apply).
- **CLA/DCO**: CLA, not DCO. No `Signed-off-by:` needed.
- **AGENTS.md**: none in the repository. `REVIEW.md` configures the
  automated "Claude Code Review" bot (`@claude review`). It is reviewer
  configuration, not instructions for contributors. I read it as a checklist
  for self-review and followed nothing else in it.
- **Text aimed at AI agents**: none in repo or issue text that tried to direct
  an agent. Issue #33498's body has a disclaimer that its root-cause and fix
  sections were LLM-produced. I treated that as data.
