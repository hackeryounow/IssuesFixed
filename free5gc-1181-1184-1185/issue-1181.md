# Issue #1181 — [Bugs] Orphaned PCC rule in postAppSessCtxProcedure when an app-session create fails after the rule is written into the SM policy decision (UMC-unsupported exit)

- Repository: free5gc/free5gc
- URL: https://github.com/free5gc/free5gc/issues/1181
- Author: ae878
- Opened: Sep 30, 2026
- State: Open

## Summary

`postAppSessCtxProcedure` builds the requested PCC rules and writes each one into the stored SM policy decision (`smPolicy.PolicyDecision.PccRules`) at L269, then continues with the sponsored-connectivity block that can fail. The failure exit at L383 — reached when `handleSponsoredConnectivityInformation` reports that usage monitoring control is not supported — returns without removing the rule and without ever creating the Application Session that would own it.

The paired release, `(*UeSmPolicyData).RemovePccRule`, is reached only through `HandleDeleteAppSessionContext`, which walks `appSession.RelatedPccRuleIds`. A rule the failed request left in the decision belongs to no app session, so nothing can release it. `GetPccRuleByAfAppId` (L255) keys the lookup on `afAppId`, so each failure with a new `afAppId` adds one rule; a repeat of the same `afAppId` reuses the existing one. Rules accumulate in the one SM policy's decision, one per distinct `afAppId`, until the PDU session (SM policy) itself is deleted.

The failure exit is taken whenever an AF requests sponsored connectivity with usage monitoring (`usgThres` present) but the negotiated SM-policy feature set does not include UMC. Only the success exit at L445 stores the app session that would make the rules releasable.

## Details

free5gc v4.2.2, PCF v1.4.3 (df6c0da).

Re-verified on the latest release: this exit is unchanged on PCF v1.4.6 (eb57557) - the revision the stock `free5gc/pcf:v4.3.0` image is built from - where the bundle's PoC measures it live (30 rejected app-session creates, each with a distinct `afAppId`, left 30 orphaned rules in the SM policy).

Acquire at L269. In the media-component loop, a rule with the requested `afAppId` is created (L257) and written into the stored decision:

```
pccRule = util.GetPccRuleByAfAppId(smPolicy.PolicyDecision.PccRules, appID)   // L255: dedup by afAppId
if pccRule == nil {
    pccRule = util.CreatePccRule(smPolicy.PccRuleIdGenerator, precedence, nil, appID)  // L257
    ...
    util.SetPccRuleRelatedData(smPolicy.PolicyDecision, pccRule, nil, &qosData, nil, nil)  // L269: decision.PccRules[id] = rule
    smPolicy.PccRuleIdGenerator++
}
```

`smPolicy.PolicyDecision` is the per-PDU-session SM policy decision, stored in `ue.SmPolicyData[smPolicyID]`; it outlives the request and lives until the SM policy (PDU session) is deleted.

Release is reached only through an app session. `HandleDeleteAppSessionContext` (L449) is the only path to `RemovePccRule`, and it iterates the app session's own rule set:

```
for _, pccRuleID := range appSession.RelatedPccRuleIds {   // L473
    if err := smPolicy.RemovePccRule(pccRuleID, &deletedSmPolicyDec); err != nil {   // L474 -> ue.go delete(decision.PccRules, id)
```

`appSession.RelatedPccRuleIds` is populated only on the success path (L400), just before the app session is stored (`AppSessionPool.Store`, L433) and the function returns nil (L445).

The failure exit after L269 does neither. With `usgThres` present, `extractUmData` succeeds, and the request proceeds into `handleSponsoredConnectivityInformation`, which rejects it when the SM policy did not negotiate UMC:

```
if err := handleSponsoredConnectivityInformation(smPolicy, relatedPccRuleIds, ascReqData.AspId,
    ascReqData.SponId, ascReqData.SponStatus, umData, &updateSMpolicy); err != nil {
    problemDetail := util.GetProblemDetail(err.Error(), util.REQUESTED_SERVICE_NOT_AUTHORIZED)
    return nil, "", &problemDetail                          // L383: rule already in decision, no app session created
}
```

```
supp := util.CheckSuppFeat(smPolicy.PolicyDecision.SuppFeat, 5) // UMC support = 5 in 29512   // L1205
if !supp {
    err := fmt.Errorf("usage Monitor Control is not supported in SMF")   // L1207
    return err
}
```

The block is entered only when the negotiated feature set has SponsoredConnectivity (bit 12) but not UMC (bit 5).

The sibling exit at L376 (`extractUmData` error, USAGE_REPORT without `usgThres`) has the identical defect, as does the earlier L365 exit ("Sponsored Connectivity not supported"), which is the one free5gc's own SMF reaches (it sends suppFeat "F", bit 12 clear) — after the same L269 acquire, so the rule leaks there too in the default configuration.

## PoC

free5gc v4.2.2, PCF v1.4.3. NRF/UDR/PCF on loopback, `oauth: false`, subscriber policy seeded in UDR, one PDU session's SM policy created per UE. No change to the leaking code. Rules are counted by reading the stored decision through the PCF's own `GET /npcf-smpolicycontrol/v1/sm-policies/{smPolicyId}` endpoint, which returns `smPolicy.PolicyDecision` — reachable state held in the SM policy. The SM policy is created with a negotiated suppFeat of `0800` — bit 12 (SponsoredConnectivity) set, bit 5 (UMC) clear — i.e. an SMF that offers sponsored connectivity but not usage monitoring.

Each request is `POST /npcf-policyauthorization/v1/app-sessions` with one media component, `aspId`/`sponId` set, and a USAGE_REPORT event with a `usgThres` present (so `extractUmData` passes and `handleSponsoredConnectivityInformation` runs and rejects at the UMC check).

The leak: one line decides it

| case | app session created? | HTTP | rules held in the SM policy decision |
| --- | --- | --- | --- |
| A, normal create then delete | yes | 201, then 204 on delete | 1, then 0 after delete |
| B, fails at L383 (this bug) | no | 403, no app session | 1, not releasable |

A shows the pair works. B is the same request against an SMF that did not negotiate UMC: the rule was already written at L269, but no app session exists to own or release it.

Accumulation — same UE and SM policy, `afAppId` distinct per request:

| requests | rules held in the SM policy decision |
| --- | --- |
| 1 | 1 |
| 5 | 5 |
| 10 | 10 |
| 15 | 15 |
| 20 | 20 |
| 25 | 25 |
| 30 | 30 |

Slope exactly 1 per request, HTTP 403 REQUESTED_SERVICE_NOT_AUTHORIZED / "usage Monitor Control is not supported in SMF" every time, no ceiling. A QosData entry is added alongside each rule.

Control — same requests but a fixed `afAppId`: `GetPccRuleByAfAppId` reuses the existing rule, so the count stays at 1. Growth needs a distinct `afAppId`, which the AF controls.

## Impact

Every app-session create that fails at L383 leaves one PCC rule (plus its QoS decision) in the SM policy's stored decision, referenced by no app session and therefore unreachable by `RemovePccRule`. The entries are held for the life of the PDU session and are freed only when the SM policy is deleted (`HandleDeleteSmPolicyContextRequest`); within a live session an AF can grow them without limit by varying `afAppId`.

Beyond memory, the orphaned rules sit in `smPolicy.PolicyDecision`, which the handler sends to the SMF on the next successful app-session update (L439-L443), so rules a request was rejected for (403) can still be provisioned toward the SMF/UPF on a later accepted request for the same PDU session. Measured on the stock `free5gc/pcf:v4.3.0` image: after the injected rejects, one accepted app-session create made the SMF log `In HandleSMPolicyUpdateNotify` followed by `apply sm policy decision error: too few fields 0` - the SMF does receive the attacker's rules inside the victim's decision (this SMF build refuses to apply them, so no UPF programming was observed). See [REPORT.md](REPORT.md) and [pcf_pcc_rule_leak_exploit.py](pcf_pcc_rule_leak_exploit.py) for the root cause, the fix process and the exploit script.

Expected behaviour: when the sponsored-connectivity block takes an error exit (L383, and likewise L376), the handler should roll back the rules it added to `smPolicy.PolicyDecision` in this call before returning, so a rejected app session leaves the decision as it found it.
