# free5gc Issue #1186 — SMF leaks the CHF charging session on the PFCP-release failure branch

Analysis target: free5gc/smf `1c9d766` (main, 2026-09-30), byte-for-byte the revision deployed as `free5gc/smf:v4.3.0` in this bundle (the image's embedded build info reports `COMMIT_HASH=1c9d7662`). The report has four parts: Root Cause, Fix Approach, CVSS 4.0 Rating, Impact.

## Root Cause

**Charging lifecycle background.** At PDU session establishment the create handler calls `CreateChargingSession` (`internal/sbi/processor/pdu_session.go:270`), which sends CHARGING_INIT to the CHF and stores the CHF-returned reference in `smContext.ChargingDataRef` (`internal/sbi/consumer/chf_service.go:162`). On the CHF side (`converged_charging.go`) a per-SUPI UE entry is created and the CDR is opened into `ue.Cdr`/`ue.Records`; **only** a paired CHARGING_RELEASE (`HandleChargingdateRelease`) ever closes it.

**The defect.** `HandlePDUSessionSMContextLocalRelease` (`pdu_session.go:1368`) releases the PFCP session first and switches on the result:

```go
pfcpResponseStatus = p.releaseSession(smContext)          // :1410

case smf_context.SessionReleaseSuccess:                   // :1414
    p.ReleaseChargingSession(smContext)                   // :1415  CHARGING_RELEASE
    ...
    p.RemoveSMContextFromAllNF(smContext, false)          //        removed from the pool

case smf_context.SessionReleaseFailed:                    // :1435
    smContext.SetState(smf_context.Active)                //        the ONLY statement in this branch
```

`releaseSession` returns `SessionReleaseFailed` whenever the UPF does not acknowledge the N4 Session Deletion — every failure path of `deletePfcpSession` (`datapath.go:777-825`: unreachable/unresolvable UPF, missing PFCP session context, send error or timeout, missing or rejected cause) folds into that value. On this outcome the handler misses **two mandatory actions at once**:

1. **No `ReleaseChargingSession`.** The function (`charging_trigger.go:64`, CHARGING_RELEASE at :72) is reachable only from the success branches, so the failed branch's charging session stays open on the CHF forever.
2. **No `RemoveSMContextFromAllNF` — and the context goes unreachable immediately.** The only caller of this handler is the create handler's duplicate-key branch (`pdu_session.go:92`), and the very next line (`:96`, `NewSMContext`) overwrites the (SUPI, PDU session ID) alias key. The SMF stores every context under **two keys** (`sm_context.go:279-284`): `smContextPool[Ref]` (the object pool, keyed by `urn:uuid:`) and `canonicalRef["supi-pid"] → Ref` (the 3GPP namespace alias). After the overwrite, the old context survives only as an object in `smContextPool`/`seidSMContextMap` — no SMF procedure can find it by SUPI or PDU session ID anymore (`GetSMContextById` resolves through `ResolveRef` to the new context), so its release can never be retried, while it keeps holding a UE IP, the PFCP session/SEIDs, TEIDs and its open `ChargingDataRef`. That is the zombie SMContext: the object is alive, its namespace is gone. The `CompareAndDelete` guard on the alias in `RemoveSMContext` (`sm_context.go:425-430`) and its comment independently acknowledge this exact takeover mechanism.

**Family gaps.** The sibling handler `HandlePDUSessionSMContextRelease` (AMF-initiated release) has the same charging gap on its failed branch (`:1317`) — there the context *is* removed, so only the CHF side is orphaned; `RemoveSMContextFromAllNF` (`sm_common.go:7`) itself never sends a charging release; and the UPF-association sweep `releaseAllResourcesOfUPF` (`association.go:279`, via `RemoveSMContextFromAllNF` at `:297`) shares the gap — it is the only reaper of zombie contexts, and even when it reaps one, the leaked CDR stays open.

## Fix Approach

1. **LocalRelease failed branch (`pdu_session.go:1435`) and the `default:` branch:** carry their own cleanup obligation. When `smContext.ChargingDataRef != ""`, call `p.ReleaseChargingSession(smContext)`, then `p.RemoveSMContextFromAllNF(smContext, false)`. Self-cleanup is mandatory here (unlike an ordinary handler) because the caller has already consumed the alias key at `:96` — no other path will ever release this context.
2. **Complementary guard in `RemoveSMContextFromAllNF` (`sm_common.go`):** before removal, send the charging release when `smContext.ChargingDataRef != ""`. This one spot also covers the sibling handler's failed branch (`:1317`) and the association sweep (`association.go:297`), making fixes 1 and 2 mutually redundant — either one alone closes the leak.
3. **Idempotence:** clear `smContext.ChargingDataRef` (under `SMLock`) once the release has been sent, and skip empty references at entry, so a success-path release followed by a later sweep cannot double-release the same reference.
4. **Optional hardening:** let the duplicate-key branch wait for the old context's release outcome, or return a meaningful error to the AMF instead of silently reaping a live session.

Scope follows the bundle workflow: only the demonstrated attack arm (the LocalRelease failure branch) plus the complementary guard; the `HandlePDUSessionSMContextUpdate` failure branch is not a leak (state reverts to Active and the context stays reachable by ref) and is out of scope. Implementation deferred until confirmed.

## CVSS 4.0 Rating

**Vector:** `CVSS:4.0/AV:A/AC:L/AT:P/PR:N/UI:N/VC:N/VI:N/VA:L/SC:N/SI:N/SA:L`
**Score:** **2.3 / 10.0 (Low)**

| Metric | Value | Grounding |
| --- | --- | --- |
| AV:A | Adjacent | the SBI binds to the core-internal segment (reference deployment br-free5gc /24); the PoC runs from a same-segment host, no internet reachability required |
| AC:L | Low | no race, no precision: inside any N4-failure window every duplicate create leaks once, linearly |
| AT:P | Precondition present | a pure-SBI attacker cannot create the N4-failure state, only wait for a UPF outage/partition window (the PoC demonstrates it via `docker pause` fault injection; the upstream harness via `POC_FORCE_RELEASE_FAILED=1`) |
| PR:N | None | the default deployment runs `oauth: false`; unauthenticated triggering verified |
| UI:N | None | pure API calls |
| VC:N/VI:N | None | no cross-user data exposure or state tampering demonstrated; the OAM probe only exposes metadata of the attacker's own session |
| VA:L | Low | per zombie the SMF holds one finite-pool UE IP, the PFCP session/SEIDs, TEIDs and memory; no crash or exhaustion demonstrated |
| SC:N/SI:N/SA:L | Low | subsequent system CHF: permanently open CDRs accumulate linearly (measured +1 per leak; upstream harness n=30); no CHF failure demonstrated |

**Score-raise conditions (not demonstrated, not scored):** inducing the N4 timeout directly from the same internal segment via PFCP flooding (topologically reachable — UPF and SMF SBI share the /24 — but not demonstrated in this PoC) → AT:N, score rises to **5.3 (Medium)**; SBI exposed beyond the core segment → AV:N, **6.3 (Medium)**; demonstrated UE address-pool/memory exhaustion → VA:H, higher still. Reverse direction: a hardened SBA with OAuth/TLS → PR:L, **2.1**. Note: the same endpoint also carries an unauthenticated "tear down any live session by (SUPI, PID)" primitive on its success branch (observed in the control runs releasing a real UE session); that effect is already folded into the VA assessment and is not scored separately.

## Impact

**Measured direct consequences per trigger (both happen at once):**

- **CHF side (unconditional):** the charging session is stranded for good — the CDR sits in CHF memory (`ue.Cdr`/`ue.Records`) and never receives its CHARGING_RELEASE. Measured open-CDR balance grows +1 per leak; sticky: UPF recovery, SMF restart and association sweeps never close it; the upstream harness measured n=1/10/30 → 1/10/30 leaked, 0 released, slope 1.
- **SMF side (unconditional):** one zombie SMContext. Detection is the dual-key mechanism of Root Cause externalized: the attacker takes the sole surviving handle from the create response's `201 Location` header and polls the read-only OAM probe `GET /nsmf-oam/v1/ue-pdu-session-info/{ref}` (server side `HandleOAMGetUEPDUSessionInfo` → `GetSMContextByRef` → bare `smContextPool.Load`) — **200 = zombie still resident, 404 = cleaned**. Measured: control (success branch) went 200→404 with a 44-byte `{"statusInfo":{"resourceStatus":"RELEASED"}}` callback and balance +0; leak (failure branch) produced no callback, no `[Termination]` log, a PFCP deletion sent and never accepted over 12.0 s, no `deleted from pool` line for `2e8ae020…` while the trigger's own new context `79f4320f…` was cleaned normally, OAM stayed 200, balance +1. The zombie keeps its UE IP, PFCP session and TEIDs. The batch variant leaks N attacker-owned sessions under one frozen window and then censuses every ref: 3/3 retained, balance +3, with the control-run refs correctly pruned as 404.

**Trigger conditions and accumulation:** requires (a) network reachability to the SMF SBI (unauthenticated in the default deployment) and (b) the N4 path to be failing at trigger time (any cause: UPF restart, crash, partition). Each unique (SUPI, PDU session ID) leaks once per session instance (the duplicate-key overwrite consumes the key); the attacker can create his own session and trigger himself (`--mode uri`/`count`, no victim UE needed) or fire at any enumerated live key. Leaks accumulate linearly with request volume inside a UPF failure window, and the leak is **conditionally triggered but unconditionally permanent**: once the failure branch runs, no later path ever sends the missing CHARGING_RELEASE.

**Not claimed:** code execution, inter-NF privilege escalation, authentication bypass, cross-user data exposure, charging-quota theft, SMF/CHF crash — none demonstrated, none scored. The severity's real substance is an unauthenticated API primitive that converts a transient N4 failure into persistent cross-NF state retention (CHF charging records + SMF resources), and the retention is not self-healing.
