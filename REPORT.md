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
| 3 | `fix/ec-keygen-order-one` | `c6436ca8f11417db69f3b17f0cbb79152e4814ac` | https://github.com/openssl/openssl/issues/33497 | fail-before (hang) / pass-after verified |

Each branch is one commit on top of the upstream base above, independent of
the others.

Suggested order if you open them one at a time: **2** (integer-overflow
hardening of widely used public lookup APIs, including a NULL dereference),
then **3** (unbounded loop in key generation on caller-supplied parameters),
then **1** (a crash in a public API on a documented use). Three open PRs is
within CONTRIBUTING's "no more than three or four".

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

**TXT_DB check** (supports the TXT_DB sentence in the commit message). A small
program reads a two-row `TXT_DB`, calls `TXT_DB_create_index()`, then
`lh_OPENSSL_STRING_doall(db->index[0], visit)`. Linked against libcrypto
without the fix:

```
index created, calling lh_OPENSSL_STRING_doall
==3522==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000000 (pc 0x000000000000 ...)
==3522==Hint: pc points to the zero page.
```

with the fix:

```
index created, calling lh_OPENSSL_STRING_doall
visited 2 rows
```

(The program is not part of the branch. The new test in `lhash_test.c` covers
the same code path.)

**Regression run:** see "Fix 1 regression run" at the end of this file.

**Competing PR search** (GitHub search over openssl/openssl, all states,
2026-11-04, searched twice): `is:pr 33496` returned nothing. Keyword searches
`OPENSSL_LH_doall`, `doall_util_fn`, `lhash thunk` and `lhash doall` found no
PR that fixes this. Hits were old merged or closed lhash work (e.g. #23192,
which introduced the thunks) and the unrelated open #33392 (new intrusive
hash table).

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

**Regression run** (ASan+UBSan, this branch): `make test TESTS="test_x509*
test_internal_x509 test_ocsp* test_cms test_req test_verify test_v3name
test_tsa test_lhash test_txt_db" HARNESS_JOBS=4` gave
`Files=18, Tests=634 ... Result: PASS`.

**Competing PR search** (all states, 2026-11-04, searched twice): `is:pr 33498`
found nothing. Keyword searches `lastpos INT_MAX`, `X509_NAME_get_index_by`,
`X509V3_get_d2i` and `OCSP_resp_find` found nothing that fixes this. The issue
reporter put a suggested fix in the issue body but has not opened a PR. If one
appears before you submit, compare: this branch also covers `OCSP_resp_find()`.

Related open PR: https://github.com/openssl/openssl/pull/33222 ("OCSP: reject
ambiguous certificate-status lookups") changes `OCSP_resp_find_status()` and
adds tests to `test/ocspapitest.c`. It does not touch `OCSP_resp_find()`.
I cherry-picked its commit (`b3ad3be9a2`) onto this branch and it applied with
no conflicts.

---

### Fix 3: `EC_KEY_generate_key()` loops forever for a group order of 1

- Branch: `fix/ec-keygen-order-one`
- SHA: `c6436ca8f11417db69f3b17f0cbb79152e4814ac`
- Files: `crypto/ec/ec_key.c`, `test/ectest.c`
- Kind: unbounded loop (100% CPU, never returns) in key generation on
  parameters an application can build with `EC_GROUP_set_generator()`.
  The issue reporter says the DER parameter parser rejects an order-1 group,
  so this is not reachable from parsed input. Hardening, not a vulnerability.
- Scope: the same loop also never ends for a group of **order 2 with
  `EC_FLAG_SM2_RANGE`** (the range bound becomes `order - 1 = 1`). The issue
  does not mention that case; the fix and test cover it. The ECDSA nonce loop in
  `crypto/ec/ecdsa_ossl.c` is already guarded by `MIN_ECDSA_SIGN_ORDERBITS`
  (64). The point-blinding loop draws modulo the field, and
  `EC_GROUP_new_curve_GFp()` requires a field above 3. Neither needs a change.
- `crypto/ec/ec_key.c` is part of the FIPS module. The "FIPS Check" CI job
  computes the new checksums and labels the PR. Contributors are not expected
  to update `providers/fips*.checksum*` by hand.

**PR title:** `ec: fail key generation when the group order leaves no valid key`

**PR body draft:**

```
ec_generate_key() draws the private key with
BN_priv_rand_range_ex(priv_key, order, ...) and retries while the result is
zero. EC_GROUP_set_generator() accepts a group order of 1, and for such a group
the only value the range can produce is 0, so EC_KEY_generate_key() spins
forever at 100% CPU. The same happens for a group of order 2 when
EC_FLAG_SM2_RANGE is set, since the bound is then order - 1.

Such groups have no cryptographic use, but they can be built through the public
API, and key generation should fail rather than hang on caller-supplied
parameters. This checks that the range bound is at least 2 before the loop and
fails with EC_R_INVALID_GROUP_ORDER otherwise. (The ECDSA nonce loop is already
protected by MIN_ECDSA_SIGN_ORDERBITS.)

A new test (degenerate_order_keygen_test in test/ectest.c) builds an order-1
group and an order-2 group with the SM2 range flag set, and checks that
EC_KEY_generate_key() fails with EC_R_INVALID_GROUP_ORDER for both. Without
the change both iterations hang.

Fixes: #33497

##### Checklist
- [x] tests are added or updated
```

**Test commands:**

```
make test TESTS=test_ec V=1
# fail-before (run directly, because it hangs):
timeout 60 ./util/wrap.pl test/ectest -test degenerate_order_keygen_test
```

**Fail-before** (`crypto/ec/ec_key.c` reverted, test kept):

```
$ time timeout 60 ./util/wrap.pl test/ectest -test degenerate_order_keygen_test
    # Subtest: degenerate_order_keygen_test
    1..2
real	1m0.011s
user	0m59.337s        <- spinning at 100% CPU until killed
exit status: 124        <- killed by timeout

$ time timeout 30 ./util/wrap.pl test/ectest -test degenerate_order_keygen_test -iter 2   # SM2 range case
real	0m30.006s
user	0m29.583s
exit status: 124
```

**Pass-after:**

```
$ timeout 60 ./util/wrap.pl test/ectest -test degenerate_order_keygen_test
    # Subtest: degenerate_order_keygen_test
    1..2
    ok 1 - iteration 1
    ok 2 - iteration 2
ok 1 - degenerate_order_keygen_test

$ make test TESTS=test_ec V=1
#     ok 20 - degenerate_order_keygen_test
../../util/wrap.pl ../../test/ectest => 0
ok 2 - running ectest
...
Result: PASS
```

**Regression run:** see "Fix 3 regression run" at the end of this file.

**Competing PR search** (all states, 2026-11-04, searched twice): `is:pr 33497`
found nothing. Keyword searches `ec_generate_key`, `EC_KEY_generate_key order`,
`infinite loop keygen`, `order of 1`, `group order 1`, `BN_priv_rand_range infinite`
found only old merged or closed PRs and unrelated open ones. The closest is open
https://github.com/openssl/openssl/pull/33465 ("Handle groups without a
generator in EC_GROUP_cmp()"), which is a different function.

**Possible follow-up (not on any branch):** the SM2 signing loop in
`crypto/sm2/sm2_sign.c` (`for (;;)` around `BN_priv_rand_range_ex(k, order, ...)`)
retries on `r == 0` with no bound, so with a degenerate order it could also
spin. I did not reproduce this. It is a separate module and would be its own PR.

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

For fix 3 (the unfixed code hangs, so run the test binary under `timeout`):

```
git checkout upstream/master -- crypto/ec/ec_key.c
make -j"$(nproc)" && timeout 60 ./util/wrap.pl test/ectest -test degenerate_order_keygen_test; echo "exit $?"   # expect exit 124
git checkout HEAD -- crypto/ec/ec_key.c
make -j"$(nproc)" && make test TESTS=test_ec V=1                                                              # expect PASS
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

## Fix 3 regression run

On `fix/ec-keygen-order-one` (ASan+UBSan):

```
$ make test TESTS="test_ec test_ecdsa test_genec test_ecparam test_internal_ec test_evp_pkey* test_pkey_check test_ecstress" HARNESS_JOBS=4
03-test_internal_ec.t ok, 15-test_ecparam.t ok, 30-test_evp_pkey_dhkem.t ok,
30-test_evp_pkey_dparam.t ok, 15-test_genec.t ok, 30-test_evp_pkey_provided.t ok,
99-test_ecstress.t ok, 91-test_pkey_check.t ok, 15-test_ecdsa.t ok, 15-test_ec.t ok
Files=10, Tests=40 ... Result: PASS

$ make test TESTS="test_internal_sm2 test_gensm2 test_internal_keymgmt test_evp_pkey_dparam test_evp" HARNESS_JOBS=4
03-test_internal_sm2.t ok, 02-test_internal_keymgmt.t ok, 15-test_gensm2.t ok,
30-test_evp_pkey_dparam.t ok, 30-test_evp.t ok
Files=5, Tests=123 ... Result: PASS
```

These two runs were on the first version of the commit (`7c3f3caf89`). The
current head `c6436ca8f1` changes only `test/ectest.c`: the negative test also
checks that the error reason is `EC_R_INVALID_GROUP_ORDER`. After that change,
`make test TESTS=test_ec V=1` gives `ok 20 - degenerate_order_keygen_test ...
Result: PASS`.

## Fix 1 regression run

On `fix/lhash-doall-null-thunk` (ASan+UBSan):

```
$ make test TESTS="test_lhash test_internal_* test_ca test_req test_x509 test_cms test_evp test_ssl_old test_sslapi test_provider* test_store test_conf*" HARNESS_JOBS=4
02-test_lhash.t ok, 80-test_ca.t ok (TXT_DB user), 25-test_req.t ok, 25-test_x509.t ok,
90-test_sslapi.t ok, 80-test_cms.t ok, 80-test_ssl_old.t ok, 30-test_evp.t ok,
90-test_store.t ok, 04-test_conf.t ok, 04-test_provider*.t ok, 30-test_provider_status.t ok,
02/03-test_internal_*.t ok (exts, ctype, context, keymgmt, provider, asn1_dsa, bn, chacha,
curve25519, ec, curve448, mdc2, ml_dsa, ml_kem, modes, namemap, poly1305,
rsa_sp800_56b, siphash, ffc, sm2, sm3, ssl_cert_table, x509, sm4, asn1)
Files=43, Tests=766 ... Result: PASS
```

## Not done in the cloud session

- No full `make test` run on any branch. Only the targeted subsets listed
  above. Run the full suite before opening each PR.
- No plain (non-sanitizer) build. Every result above comes from
  `-d enable-asan enable-ubsan --strict-warnings` (which also confirms the
  changes compile cleanly with `-Werror`).
- No `no-deprecated` build or other configuration variants. The new ectest
  test uses `EC_KEY_*` the same way the existing, unguarded
  `check_ec_key_field_public_range_test` does.
