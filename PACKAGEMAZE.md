# Plane PackageMaze demo

This clone uses the PackageMaze Organization's [demo-npm](https://www.packagemaze.com/packagemaze/demo-npm) and [demo-pypi](https://www.packagemaze.com/packagemaze/demo-pypi) Feeds.

Run commands from the repository root:

```sh
./with-packagemaze pnpm install --frozen-lockfile --store-dir .secrets/packagemaze/pnpm-store
./with-packagemaze uv venv --python 3.12 apps/api/.venv
./with-packagemaze uv pip install --python apps/api/.venv/bin/python \
  -r apps/api/requirements/local.txt \
  -r apps/api/requirements/production.txt \
  -r apps/api/requirements/test.txt
./with-packagemaze uv pip check --python apps/api/.venv/bin/python
```

The wrapper supplies authentication only to the command it runs. The newly issued read-only Token covers only the two demo Feeds and expires after 30 days. It is stored in the ignored, owner-only file `.secrets/packagemaze/token`; a temporary trusted npm user config is created for each invocation and removed on exit. An exported `MAZE_TOKEN` overrides the stored Token. No global npm, pip, or uv configuration is changed.

The npm lockfile and the repository's dependency build-script allowlist are preserved. Python requirements include local, production, and test dependencies. `requirements/base.txt` includes the shared constraints file, so local installs, API CI, and Docker use the same verified dependency versions. Constraints restrict selected packages; they do not install test or development dependencies in a production-only install. Update the constraints deliberately with dependency changes and verify compatibility on each target platform. On this Mac, the existing Homebrew libpq supplies `pg_config` for psycopg-c.

GitHub Actions uses `packagemaze/setup-maze@v0.0.5` to exchange GitHub OIDC identity for short-lived read-only Tokens. The web, i18n, and React Doctor jobs use demo-npm; API jobs use demo-pypi. Matching CI access rules are scoped to `allwhat/plane`. Docker builds use the action’s shell bundles through required BuildKit secret mounts; credentials are removed when each RUN finishes. Manual web runs build the full workspace, and the Docker rehearsal builds complete web and API images without pushing them. Docker images, OS packages, and Python runtimes are outside these npm/PyPI Feeds.

Feed policies may hold a package version. Use PackageMaze diagnostics instead of adding a second index, changing artifact hashes, or bypassing the Feed.

## Reproducible Python installs

The initial cold comparison crossed the 24-hour Minimum Age boundary for `python-dateutil 3.0.0` (uploaded 21 October at 23:41 UTC). uv tried it, then downloaded older botocore candidates before returning to `python-dateutil 2.9.0.post0`, because the selected boto3/botocore line requires dateutil below 3. The final installed versions alone did not capture that extra resolver work.

The constraints file preserves all 124 versions from the successful baseline. With fresh uv caches and environments on the same Mac, three constrained installs took 38.33, 34.66 and 34.75 seconds, all passing `uv pip check`. These measurements used uv 0.11.2 and CPython 3.12.13 through `demo-pypi`. The Feed retained server-side data; only the client caches were cold. The constraints do not replace Feed policy, artifact integrity checks, or an isolated build environment's own build requirements.

## Verified result and timings — 23 October 2026

All 20 pnpm workspace projects installed successfully, adding 1,493 package instances. The Python 3.12 environment contains all 124 distributions from the combined local, production, and test requirements. `uv pip check`, Django/psycopg/pytest/native imports, Turbo and oxfmt execution, React/React Router imports, and a real sharp native image operation passed. The lockfile, package manifests, workspace catalog, and Python requirements are unchanged. PackageMaze recorded completed installs on both demo Feeds using the new read-only Token. The local setup review returned no warning or blocking diagnostics.

| Install scenario                             | npm / pnpm 11.3.0 |     Python / uv 0.11.2 |
| -------------------------------------------- | ----------------: | ---------------------: |
| Fresh clone and empty package caches         |          7m 48.1s | approximately 1m 44.4s |
| Clean destination with populated caches      |             6.39s |                 39.52s |
| Repeat into an already installed destination |             0.23s |                  0.15s |

The npm cold duration is pnpm's reported elapsed time. Python cold timing is the observed log creation-to-completion interval; uv separately reported 78 seconds resolving, 25.09 seconds preparing, and 226 ms installing. Subsequent runs were timed around process execution with a monotonic clock. The first Python cache-backed clean install took 43.82 seconds while npm was active; the later 39.52-second run was after the other installs completed, with 39.08 seconds resolving and 282 ms installing.

The npm cold run spent 151.4 seconds checking 1,739 lockfile entries against pnpm's supply-chain policies before its ordinary install phase. The pinned client defaults to a 24-hour release-age check; the policy was preserved. The clean cache-backed npm run used a separate source archive with the same manifests and store. Its root Husky prepare step had no Git checkout to configure, so it is a dependency-install comparison rather than a complete clone bootstrap.

## Performance assessment

Repeat developer installs are fast. npm can also reconstruct a clean install quickly from its local store. Cold npm acquisition and Python resolution warrant profiling before claiming consistently fast setup.

During the cold installation, three requests for an already requested 1,506-byte npm tarball took 2.05–2.92 seconds; React metadata took 3.18–4.58 seconds. After completion and cache warming, the same npm endpoint samples generally took 32–79 ms after the first connection setup. These measurements differ in both load and cache state, so they do not establish which one caused the improvement.

Python's Django Simple metadata requests still took 0.97–1.05 seconds after the initial request. A clean environment with populated caches still spent about 39 seconds resolving. uv logged repeated `Cached response doesn't match current request` warnings for wheel metadata. Those warnings are evidence to investigate, not a proven cause.

Read-only Cloudflare queries scoped to the demo Feeds confirmed server latency on production Worker version `1cc34139-a823-4086-9c18-0f30f455f5de`. During the isolated Python warm run, recorded top-level request durations had a median of 2,290 ms and p95 of 3,450 ms. GET requests returning 304 had a median of 2,188 ms; HEAD requests returning 304 had a median of 2,260 ms. One HEAD/304 took 2,532 ms, followed by a separate 60 ms usage-journal stage. These are server operation durations, not client transfer timings. Cloudflare aggregate sampling means request totals and percentiles are not a complete request census.

The next useful investigation is the Python Simple and wheel-metadata cache-validation path: identify why a response with no changed body still spends about two seconds in dispatch, then split authentication, policy, storage, upstream validation, and rendering time using existing sampled traces. No native trace spans were returned for the example request, so an internal D1/R2/upstream bottleneck has not been established. Production configuration and code were not changed.

## Evidence and limits

Measurements ran on this Apple Silicon Mac on 23 October 2026, approximately 08:14–08:25 AEDT, from Plane commit `0c60e7d337a1890a15ade2b06ef6479dca6d29e0`. The first npm and Python installs overlapped, and the cold npm interval includes a small request probe and the first Python repeat. There is no controlled direct-upstream baseline or agreed performance budget, so these results establish observed latency and profiling priorities rather than an SLA pass/fail.

Sanitized install logs, timed-run JSON, bounded HTTP samples, and Cloudflare summaries are in `tmp/packagemaze-demo/` (ignored by Git). `server-profile.json` includes the setup review and completed-install evidence for both Feeds. Run the request sampler only with the scoped local Token; do not send credentials to another registry.

The Token expires on **22 November 2026 at 08:11 AEDT** (`2026-11-21T21:11:04Z`). Refresh it through PackageMaze when needed. The timings above precede the CI migration on `codex/plane-demo-ci`; CI validation is recorded in the pull request.

Client configuration references: [pnpm settings](https://pnpm.io/settings) and [uv package indexes](https://docs.astral.sh/uv/concepts/indexes/). uv's named default index excludes the public PyPI fallback; its credentials are supplied only by the wrapper.
