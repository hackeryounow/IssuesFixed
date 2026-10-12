# Issue #1186 — [Bugs] SMF: local release leaves a charging session open on the CHF when the PFCP session deletion fails

- Repository: free5gc/free5gc
- URL: https://github.com/free5gc/free5gc/issues/1186
- Author: ae878
- Opened: Oct 3, 2026
- State: Open

## Bug Description

To set up a PDU session, the SMF opens a charging session on the CHF: `CreateChargingSession` sends CHARGING_INIT and remembers the returned charging reference (`smContext.ChargingDataRef`). When such a session is torn down locally — `HandlePDUSessionSMContextLocalRelease`, e.g. a duplicate PDU session request for the same key — the SMF should close that charging session with CHARGING_RELEASE. On the failed-PFCP branch it does not, and it also does not remove the SMContext, so the session is left half-released with its charging session open on the CHF.

The local-release handler releases the PFCP session (`releaseSession`) and switches on the result. `ReleaseChargingSession` (CHARGING_RELEASE) is called only on `SessionReleaseSuccess`. On `SessionReleaseFailed` the handler only sets the state back to Active and returns — no charging release, and no `RemoveSMContextFromAllNF`.

Source analysis on SMF v1.4.3, commit 2232d299. Still present on upstream main (1c9d766, 2026-09-30): in `HandlePDUSessionSMContextLocalRelease` the `SessionReleaseSuccess` branch still calls `ReleaseChargingSession` (pdu_session.go:1415) while the `SessionReleaseFailed` branch (pdu_session.go:1435) still only reverts the state — no charging release and no `RemoveSMContextFromAllNF`. (The `HandlePDUSessionSMContextRelease` handler has the same charging gap on its `SessionReleaseFailed` branch, at 1311/1317; `RemoveSMContextFromAllNF` in sm_common.go:7 also still has no charging release. Note: `HandlePDUSessionSMContextUpdate`'s failed branch is not a leak — it reverts the context to Active and leaves it reachable by smContextRef, so a later release can still close it.)

### The charging session is opened at create and its reference stored

[pdu_session.go L249-L252 (2232d299)](https://github.com/free5gc/smf/blob/2232d2992f204207e58830ac8ee55bf0cd832f92/internal/sbi/processor/pdu_session.go#L249-L252) — `CreateChargingSession` sends CHARGING_INIT and stores the reference the CHF returns in `smContext.ChargingDataRef`.

### Local release switches on the PFCP result; only success releases charging

[pdu_session.go L1340-L1386 (2232d299)](https://github.com/free5gc/smf/blob/2232d2992f204207e58830ac8ee55bf0cd832f92/internal/sbi/processor/pdu_session.go#L1340-L1386):

```
pfcpResponseStatus := releaseSession(smContext)
switch pfcpResponseStatus {
case smf_context.SessionReleaseSuccess:
    p.ReleaseChargingSession(smContext)        // :1355-1356  CHARGING_RELEASE to the CHF
    smContext.SetState(smf_context.InActivePending)
    ...
    p.RemoveSMContextFromAllNF(smContext, false)

case smf_context.SessionReleaseFailed:         // :1376-1378  PFCP delete failed / UPF unreachable
    smContext.SetState(smf_context.Active)     // only reverts state
    // <-- no ReleaseChargingSession, no RemoveSMContextFromAllNF
}
```

### The charging release exists only on the success paths

[charging_trigger.go L64-L68 (2232d299)](https://github.com/free5gc/smf/blob/2232d2992f204207e58830ac8ee55bf0cd832f92/internal/sbi/processor/charging_trigger.go#L64-L68) — `ReleaseChargingSession` sends CHARGING_RELEASE. It is reached only from the `SessionReleaseSuccess` branches. The failed-PFCP branch never calls it.

### What makes the PFCP delete fail

`releaseSession` returns `SessionReleaseFailed` when the UPF does not acknowledge the PFCP session deletion (unreachable UPF, or a rejected/timed-out N4 Session Deletion). The charging session has already been opened at that point, so any such failure leaves it open on the CHF.

## To Reproduce

On SMF v1.4.3, driven through the project's own SMF unit-test harness (internal/sbi/processor/pdu_session_test.go): gock-stubbed NRF/UDM/PCF/CHF and a stub PFCP, no real UPF.

How charging is opened: the NRF stub lets the SMF discover a CHF, so CHFSelection succeeds and `CreateChargingSession` sends CHARGING_INIT; the CHF stub answers 201 with a charging reference.

How the PFCP release is made to fail: the failed-PFCP outcome is reproduced at its exact point with a one-line, env-gated fault in `releaseSession` (`POC_FORCE_RELEASE_FAILED=1` → returns `SessionReleaseFailed`), which is the value a real UPF-unreachable / rejected N4 deletion yields. The leaking code under test (the `SessionReleaseFailed` branch that omits `ReleaseChargingSession`) is unchanged and off by default.

What is counted: the CHF stub counts CHARGING_INIT (`POST .../chargingdata`) and CHARGING_RELEASE (`POST .../chargingdata/{ref}/release`) with `gock.Observe`; SMContext liveness is read with `GetSMContextById(supi, pid)`.

| case | CHARGING_INIT sent | CHARGING_RELEASE sent | live SMContexts | live ChargingDataRef |
| --- | --- | --- | --- | --- |
| control (PFCP OK) | 1 | 1 | 0 | 0 |
| LEAK n=1 | 1 | 0 | 1 | 1 |
| LEAK n=10 | 10 | 0 | 10 | 10 |
| LEAK n=30 | 30 | 0 | 30 | 30 |

### The leak

- Each of the 30 local releases opened a charging session and then hit `SessionReleaseFailed`. Not one CHARGING_RELEASE reached the CHF.
- The SMContext is also not removed on this branch, and — crucially — it becomes immediately unreachable within the same request: LocalRelease is invoked from the create handler for a duplicate key (pdu_session.go:73-75, main 91-93), and the very next line, `NewSMContext` (pdu_session.go:78, main :96), overwrites canonicalRef with the new context. On the failed branch the old SMContext is therefore no longer findable by SUPI / PDU Session ID, while it stays held in `smContextPool` and `seidSMContextMap` as a zombie, so no SMF procedure can reach it by SUPI / PDU Session ID.
- The only remaining path is a UPF association release (`releaseAllResourcesOfUPF`, association.go:163, main :279), which ranges smContextPool and may remove the context via `RemoveSMContextFromAllNF` — but that function sends no CHARGING_RELEASE, so the charging session stays open on the CHF either way. If the UPF stays associated (e.g. it rejected the N4 deletion rather than going unreachable), the zombie SMContext itself also persists.
- Accumulation: the charging session leaks on the CHF once per failed local release (slope 1), regardless of whether the zombie SMContext is later swept; bounded at 1 only when the exact same session is retried.

## Expected Behavior

The `SessionReleaseFailed` (and default) branch of `HandlePDUSessionSMContextLocalRelease` does not call `RemoveSMContextFromAllNF`, so it must handle cleanup itself: on that branch the SMF should release the charging session (and finish removing the now-unreachable SMContext) before it stops, so a failed PFCP deletion does not leave a charging session open on the CHF together with a zombie SMContext.

(Adding a charging release inside `RemoveSMContextFromAllNF` when `smContext.ChargingDataRef != ""` is a separate, complementary fix — it covers the paths that do call `RemoveSMContextFromAllNF`, i.e. the `SessionReleaseFailed` branch of the `HandlePDUSessionSMContextRelease` handler. It does not fix this LocalRelease bug on its own, because this branch never reaches that function.)

## Environment

- free5GC Version: SMF v1.4.3 (2232d299); reproduced through the SMF unit-test harness, not a full deployment
- OS: Ubuntu 24.04.5 LTS
- Kernel version: 6.8.0-139-generic
- go version: go1.25.5 linux/amd64

## Log File

```
[INFO][SMF][Charging] Send Charging Data Request[Init] successfully
[INFO][SMF][PduSess] In HandlePDUSessionSMContextLocalRelease
[TRAC][SMF][CTX] In case SessionReleaseFailed
```

The init is sent; the local release takes the `SessionReleaseFailed` branch; no charging release is logged, and no SMContext removal is logged on this branch.

## Additional Context

This is a different bug from the create-time charging leak (e.g. an ApplySessionRules failure): there, the full cleanup runs and only the charging release is missing. Here, on `SessionReleaseFailed` the handler does not even run the cleanup — it reverts the state and returns — so both the SMContext and its charging session are left live. The sibling handler `HandlePDUSessionSMContextRelease` (AMF-initiated release) has the same charging gap on its failed branch, but there the SMContext is removed, so only the CHF-side charging session is orphaned.

Anticipated objection: one might argue that if the PFCP deletion failed the UPF session is still live, so keeping the charging session open is correct. That does not hold here. Because the old SMContext is overwritten in the same create request (above), no SMF procedure can reach it by SUPI / PDU Session ID to retry the release. The one path that can still sweep it is a UPF association release (`releaseAllResourcesOfUPF`), which — in the "UPF unreachable" case — likely runs after a heartbeat failure and removes the context via `RemoveSMContextFromAllNF`; but that function sends no CHARGING_RELEASE, so the charging session stays open on the CHF either way. Only the zombie SMContext is conditional: it is swept if the association drops, and persists if the UPF stays associated (it merely rejected the N4 deletion). The charging leak is unconditional, which is why the failed branch must release charging itself.

## How did we find?

We found this bug by using our (work-in-progress) static analysis tool, manually verified it, and made a runnable PoC.
