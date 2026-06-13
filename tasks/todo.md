# Wax Bugs Audit

## Plan
- [x] Inspect the WAL replay and open/recovery path for corruption handling.
- [x] Identify a high-confidence production correctness bug in pending WAL replay.
- [x] Patch the bug with the smallest safe change.
- [x] Add a regression test that exercises the failure mode directly.
- [x] Document the outcome and remaining verification gap.

## Review
- Fixed `WALRingReader.scanPendingMutationsWithState` so checksum-valid but undecodable WAL entries now throw corruption instead of being skipped.
- Added `Tests/WaxCoreTests/WALRingReaderTests.swift` to lock the regression down.
- Verification gap: command execution was unavailable in this run, so `swift test` and trait builds could not be executed locally.