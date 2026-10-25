# spacetimedb-swift, with MeNIN's fixes

This is a fork of [avias8/spacetimedb-swift](https://github.com/avias8/spacetimedb-swift), the community Swift SDK for SpacetimeDB, at **v0.23.0 (`994b0d4`)**. That repo is not a Clockwork Labs SDK: Clockwork's monorepo has no Swift SDK. avias8/spacetimedb-swift mirrors `sdks/swift` from the `swift-integration` branch of avias8's SpacetimeDB fork. The SDK speaks the `v2.bsatn.spacetimedb` WebSocket protocol, and we run it against SpacetimeDB 2.13.

MeNIN (Cal Hacks 13.0) vendored its `Sources/` into the iOS app at `ios/Vendor/SpacetimeDB` and fixed three bugs that showed up on real phones. This fork is upstream v0.23.0 with those fixes on top as four commits: three fixes and their tests. The fixes go back upstream as a draft PR (see `PR.md`).

## License

Unchanged: **SpacetimeDB Business Source License 1.1** (`LICENSE.txt`). The licensor is Clockwork Laboratories, Inc. and the Change License is AGPL v3.0 with a linking exception. Our copy of `LICENSE.txt` is byte-for-byte the file in upstream `994b0d4`. Nothing here is relicensed. GitHub shows the license as "NOASSERTION" because it doesn't recognise BSL.

## The fixes

1. **No implicit subscribe-all on connect** (`0001`).
   - Symptom and cause: every phone received every row of every registered table, including other rooms' 20 Hz prosody stream, because each connect sent a handle-less `SELECT * FROM <table>` for every registered table, beside the app's own filtered subscriptions.
   - Fix: the app's subscriptions alone decide what the cache holds. `subscribeAll(tables:)` stays public for apps that want the old behaviour.
2. **`rows` lists each distinct row once** (`0002`).
   - Symptom and cause: a row covered by two overlapping subscriptions showed up twice in `TableCache.rows`, because `snapshot()` treated the per-row refcount as a multiplicity.
   - Fix: list each entry with count > 0 once and keep the refcount, as the TypeScript SDK's `iter()` does.
3. **A reconnect replaces each table's rows** (`0003`).
   - Symptom and cause: after a network drop, a phone stayed on a room's old screen until it was killed. The replayed subscription rows were added on top of the old cache, so a changed row existed twice (old and new) and a deleted one never left.
   - Fix: on each connect, the first subscription rows each table receives replace that table's cache, in the same pass. The new `ClientCache.clear(tables:)` does the clearing.

`0004` adds tests for 0002 and 0003, and updates the one upstream test that asserted duplicate rows.

Not upstreamed: our vendored copy also trims `Package.swift` to the library target and leaves out Tests, Benchmarks and DocC. That is packaging, not a fix, so this fork keeps upstream's full tree.

## How it was tested

- **Real iPhones, in production use.** Fixes 1 and 2 have been in MeNIN's iOS app since its first checkpoint (Oct 23). Fix 3 shipped in build 31 (Oct 24) and has been in every build since, through build 45, the judging build. Two iPhones per room, talking to SpacetimeDB 2.13 on Maincloud.
- **The 10:49 PM reconnect proof (Sat Oct 24).** In a live room on build 31, airplane mode was switched on and off on a real iPhone. The phone came back on the right screen with every player listed, without a relaunch. Before build 31, phones got stuck on a room's old screen after drops. On build 44, five more airplane-mode reconnects came back cleanly in 6–21 s, checked from the server's side.
- **A drop-and-replay harness, with a control.** A macOS command-line client ran on a copy of the SDK against a local SpacetimeDB 2.13. It used one test-only hook that drops the socket, which is not part of these patches.
  - The client joins a room and holds two overlapping subscriptions on `player`. Then the socket is dropped, and while it is down a second client joins and the room's phase changes.
  - With fix 3, two runs ended with 1 room row (the new phase) and the players `[Ada, Bo]`, each listed once. The overlap shows fix 2 working.
  - With fix 3 switched off, the control run ended with **2 room rows, `dealing` and `lobby`**. That is the ghost.
- **hall-soak** (`spacetime/scripts/hall-soak.ts` in MeNIN): five battles on real websockets, with drops and rejoins. It drives the server module through the **TypeScript** SDK. It shows the server's side of a rejoin is right, but it does not run this Swift code.
- **Unit tests.** `swift test` on this branch: 86 tests, 0 failures, 4 skipped (upstream's live-server tests, which need `SPACETIMEDB_SWIFT_LIVE_TESTS=1`). Run Oct 25 on macOS with the Swift 6 toolchain.
- **Not tested yet:**
  - visionOS and watchOS;
  - the `swift-integration` copy, beyond applying the patches. It has the same code, and all four patches apply cleanly to it with `git apply --directory=sdks/swift` (checked Oct 25).

## Making the fork

```bash
gh repo fork avias8/spacetimedb-swift --clone   # keeps history, the license and the PR base
cd spacetimedb-swift
git switch -c menin-fixes v0.23.0
for p in /path/to/sdk-upstream/000*.patch; do
  git apply --index "$p"
  git commit -m "$(sed -n 's/^Subject: //p' "$p" | head -1)"
done
swift test
```

Each patch file starts with its commit message (a `Subject:` line and a body) and then a plain `git diff` against the previous patch. `git apply` skips the text before the first `diff --git` line. The series applies cleanly, in order, to a fresh `v0.23.0` checkout. Fixes 1–3 contain exactly the code MeNIN ships; only the comments differ, reworded for upstream with our `HOMER PATCH` markers removed. `vendored-vs-v0.23.0.diff` is the exact diff of our vendored copy, markers included, for comparison.
