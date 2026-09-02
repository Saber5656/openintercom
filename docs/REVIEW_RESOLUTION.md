# Review resolution record

- Repository: `Saber5656/openintercom`
- Pull request: #1
- Parent head observed before this addendum: `bb6f03c6d0394740f43a46dd19c29b239734681a`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNkIZc6Onlqu`

### Fail closed on insecure key permissions

- Finding: The existing review thread `PRRT_kwDOTNkIZc6Onlqu` identifies this contract gap.
- Normative resolution: Treat CA/server private-key permissions other than `0600` (and unsafe ownership) as a startup error; never continue with a warning-only path.
- Focused verification before resolving this thread: Set group/world-readable key permissions and assert hub startup refuses to serve.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXL`

### Include localhost in the CA constraints

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXL` identifies this contract gap.
- Normative resolution: Include every shipped localhost DNS SAN in the critical CA NameConstraints, alongside the permitted `.local`/hostname policy, and keep certificate SANs within that set.
- Focused verification before resolving this thread: Issue a certificate for `localhost` and an out-of-constraint name and assert only the former validates.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXO`

### Drop HSTS while the HTTP setup listener is required

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXO` identifies this contract gap.
- Normative resolution: Do not emit a host-wide HSTS policy that can upgrade the required HTTP setup endpoint; enable scoped HSTS only after HTTPS operation and setup lifecycle make it safe.
- Focused verification before resolving this thread: Visit setup before and after HTTPS activation and assert HTTP setup remains reachable when required and HTTPS responses have the documented policy.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXS`

### Allow ephemeral listener ports for test harnesses

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXS` identifies this contract gap.
- Normative resolution: Permit `127.0.0.1:0` only in an explicit test/harness configuration and return the bound port; retain production validation that requires configured non-ephemeral listeners.
- Focused verification before resolving this thread: Start isolated HTTP/HTTPS test hubs with port zero and assert they report usable bound ports while production config rejects it.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXX`

### Set the admin password before starting the hub

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXX` identifies this contract gap.
- Normative resolution: Complete offline store initialization and `admin set-password` before starting `serve`, so the setup sequence never needs a second writer while the bbolt lock is held.
- Focused verification before resolving this thread: Run a fresh setup and assert the password is established before serve acquires the store lock.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXa`

### Add an active timestamp to call.state

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXa` identifies this contract gap.
- Normative resolution: Add an authoritative `answered_at`/`active_since` timestamp to `call.state` and resync payloads; duration is derived from it and remains available after reload.
- Focused verification before resolving this thread: Answer a call, reload/resync, and assert active duration uses the transmitted timestamp rather than local start time.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXe`

### Return an IP-based pairing URL for Android

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXe` identifies this contract gap.
- Normative resolution: Return an explicit IP-based pairing URL, preferably first or separately labeled, in addition to the best-name URL so Android setup does not depend on `.local` resolution.
- Focused verification before resolving this thread: Generate pairing output on a LAN fixture and assert a reachable IP URL is present and clearly labeled for Android.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXh`

### Avoid duration math before a call is active

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXh` identifies this contract gap.
- Normative resolution: Compute duration only when `AnsweredAt` exists; connecting-to-ended calls carry null/zero duration and a distinct pre-answer end reason.
- Focused verification before resolving this thread: Hang up during connecting and after answer and assert the two records have the correct duration semantics.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXl`

### Create a writable /data before switching to nonroot

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXl` identifies this contract gap.
- Normative resolution: Create `/data` in the image, assign ownership to the runtime nonroot user before the process switch, and verify database/CA subdirectories are writable at startup.
- Focused verification before resolving this thread: Build and run the container as nonroot with a fresh volume and assert it can initialize and persist both bbolt and CA files.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXo`

### Run npm audit in the web CI job

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXo` identifies this contract gap.
- Normative resolution: Add the required high-severity `npm audit` gate to the web job after dependency installation, with its failure policy explicit and reproducible.
- Focused verification before resolving this thread: Run the workflow parser/CI fixture and assert the web job executes the high-severity audit and fails on a high vulnerability.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkIZc6OnrXr`

### Keep the station service worker from controlling /admin

- Finding: The existing review thread `PRRT_kwDOTNkIZc6OnrXr` identifies this contract gap.
- Normative resolution: Register the station worker under a `/station/` scope/path and set an explicit scope so it cannot claim `/admin` or other hub routes.
- Focused verification before resolving this thread: Load station and admin paths with the worker installed and assert only station URLs are controlled.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.