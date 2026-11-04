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
| 2 | `fix/x509-cursor-int-max` | `d400d7558f862ea3b4a044a4de1287cc90d3933b` | https://github.com/openssl/openssl/issues/33498 | fail-before / pass-after verified |

Suggested order if you open them one at a time: **2** (integer-overflow
hardening of widely used public lookup APIs, including a NULL dereference),
then **1** (a crash in a public API on a documented use).

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

### Fix 2: lookup cursors overflow at `lastpos == INT_MAX` (x509 and OCSP)

- Branch: `fix/x509-cursor-int-max`
- SHA: `d400d7558f862ea3b4a044a4de1287cc90d3933b`
- Files: `crypto/x509/x509name.c`, `crypto/x509/x509_v3.c`,
  `crypto/x509/x509_att.c`, `crypto/x509/v3_lib.c`, `crypto/ocsp/ocsp_cl.c`,
  `test/x509_internal_test.c`, `test/ocspapitest.c`
- Kind: signed integer overflow (UB) in public API cursors, which leads to a
  NULL dereference (`X509_NAME_get_index_by_*`, `OCSP_resp_find`) or a wrong
  result (others). The cursor comes from the calling application, not from
  parsed input, so this is hardening, not a vulnerability.
- Scope: I grepped `crypto/` for the `lastpos++` / `*idx + 1` / `last++` cursor
  pattern. The issue lists the four x509 sites. `OCSP_resp_find()` has the same
  bug and is **not** in the issue; it is fixed in the same commit. Every other
  `*_get_*_by_*` cursor function is a wrapper around one of these (checked).

**PR title:** `x509, ocsp: avoid lookup cursor overflow at INT_MAX`

**PR body draft:**

```
The lookup functions that take a "last position" cursor search the entries
after it and return -1 (or NULL with *idx = -1) when there are none. They all
compute lastpos + 1 before comparing it with the number of entries, which is
signed overflow when the caller passes INT_MAX:

- X509_NAME_get_index_by_OBJ() clamps negatives before the increment, so the
  wrapped INT_MIN reaches sk_X509_NAME_ENTRY_value(), which returns NULL, and
  the entry is dereferenced (crash, even with one entry).
- X509v3_get_ext_by_OBJ(), X509v3_get_ext_by_critical() and
  X509at_get_attr_by_OBJ() clamp after the increment, so the search restarts
  at 0 and returns the first match instead of -1.
- X509V3_get_d2i() does the same with *idx + 1 and returns a decoded extension
  with *idx = 0 instead of NULL / *idx = -1.
- OCSP_resp_find() increments `last` the same way and dereferences the NULL
  single response. (Not in the issue; same pattern.)

The fix returns "not found" as soon as the cursor is at or past the last
entry. The count is an int, so lastpos + 1 cannot overflow after that check.
Results for every other cursor value are unchanged.

Tests: test_lookup_lastpos_int_max (x509_internal_test) covers
X509_NAME_get_index_by_NID, X509_get_ext_by_NID, X509_get_ext_by_critical,
X509_REQ_get_attr_by_NID and X509_get_ext_d2i; test_resp_find_last_int_max
(ocspapitest) covers OCSP_resp_find. Both fail on master under UBSan and pass
with the change.

Fixes: #33498

##### Checklist
- [x] tests are added or updated
```

**Test commands:**

```
make test TESTS="test_internal_x509 test_ocsp" V=1
```

**Fail-before** (library changes reverted, tests kept; ASan+UBSan build with
`-fno-sanitize-recover=all`, so each test binary stops at the first overflow):

```
../openssl/crypto/x509/x509name.c:90:17: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
../../util/wrap.pl ../../test/x509_internal_test => 1
not ok 1 - running x509_internal_test
...
../openssl/crypto/ocsp/ocsp_cl.c:227:13: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
../../util/wrap.pl ../../test/ocspapitest ... => 1
    not ok 1 - running ocspapitest
not ok 12 - === OCSP API TESTS===
Result: FAIL
```

Reverting one file at a time (other fixes kept) shows each site fails on its own:

```
== only crypto/x509/x509name.c reverted:
../openssl/crypto/x509/x509name.c:90:17: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
== only crypto/x509/x509_v3.c reverted:
../openssl/crypto/x509/x509_v3.c:51:12: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
== only crypto/x509/x509_att.c reverted:
../openssl/crypto/x509/x509_att.c:44:12: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
== only crypto/x509/v3_lib.c reverted:
../openssl/crypto/x509/v3_lib.c:216:17: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
```

The issue reports that without sanitizers the name lookup crashes with SIGSEGV
and the ext/attr/d2i lookups return 0 / an object, which the new
`TEST_int_eq(..., -1)` checks would catch. I only ran the sanitizer build in the
cloud session, so check this in your plain `./Configure -d` build too.

**Pass-after:**

```
#     ok 7 - test_lookup_lastpos_int_max
../../util/wrap.pl ../../test/x509_internal_test => 0
ok 1 - running x509_internal_test
...
#         ok 1 - test_resp_find_last_int_max
    ok 1 - running ocspapitest
ok 12 - === OCSP API TESTS===
Result: PASS
```

**Competing PR search** (all states, 2026-11-04): `is:pr 33498` found nothing.
Keyword searches `lastpos INT_MAX`, `X509_NAME_get_index_by` and
`X509V3_get_d2i` (PRs since 2026-05-01) found nothing related. The issue
reporter put a suggested fix in the issue body but has not opened a PR. If one
appears before you submit, compare: this branch also covers `OCSP_resp_find()`.

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

For fix 2:

```
git checkout upstream/master -- crypto/x509 crypto/ocsp
make -j"$(nproc)" && make test TESTS="test_internal_x509 test_ocsp" V=1   # expect FAIL
git checkout HEAD -- crypto/x509 crypto/ocsp
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
