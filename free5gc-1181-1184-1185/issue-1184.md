# Issue #1184 — [Bugs] PCF: app-session create leaves an orphaned PCC rule in the SM policy decision when extractUmData fails

- Repository: free5gc/free5gc
- URL: https://github.com/free5gc/free5gc/issues/1184
- Author: ae878
- Opened: Oct 3, 2026
- State: Open

## Bug Description

To authorize an AF application session, the PCF's `postAppSessCtxProcedure` builds the requested PCC rules and writes each one into the stored SM policy decision (`smPolicy.PolicyDecision.PccRules`) before it has created the Application Session that is supposed to own them. If a later step in the same handler fails, the PCF returns without creating the app session and without removing the rule it already wrote. The rule is then owned by no app session, and the release path can never reach it.

The paired release, `(*UeSmPolicyData).RemovePccRule`, is reached through `HandleDeleteAppSessionContext`, which walks `appSession.RelatedPccRuleIds`. A rule the failed request left in the decision belongs to no app session, so nothing can release it. Rules accumulate in the one SM policy's decision, one per distinct `afAppId`, until the PDU session (SM policy) itself is deleted.

The failure exit here sits behind `extractUmData` and is taken whenever an AF asks for a USAGE_REPORT event subscription without a `usgThres`. Only the success exit stores the app session that would make the rules releasable.

Source analysis on PCF v1.4.3, commit df6c0da. Still present on upstream main (eb57557, 2026-08-20): the rule is still written into the decision (`policyauthorization.go:270`), the `extractUmData` error exit still returns without removing it (`:375`), and the rule id is still registered into `appSession.RelatedPccRuleIds` only on the success path (`:401`) — so the delete-time walk still cannot reach a rule left by a failed create. Measured live on the stock `free5gc/pcf:v4.3.0` image (which is built from eb57557): 30 rejected app-session creates left 30 orphaned rules in the stored decision.

Same root cause as #1181, but a different trigger: #1181 needs an SMF that did not negotiate UMC, while this exit (L376) is hit even when UMC is negotiated (the PoC uses suppFeat `0fff`). A fix scoped to the UMC check would not close this one.

### The rule is written into the stored decision first.

In the media-component loop, a rule with the requested `afAppId` is created and written into the stored decision:

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

### The failure exit after the write removes nothing and creates no app session.

The sponsored-connectivity block runs after the rule is already in the decision, and `extractUmData` returns an error for a USAGE_REPORT subscription with no `usgThres`:

```
if tempUmData, err := extractUmData(umID, eventSubs, ascReqData.EvSubsc.UsgThres); err != nil {
    problemDetail := util.GetProblemDetail(err.Error(), util.REQUESTED_SERVICE_NOT_AUTHORIZED)
    return nil, "", &problemDetail                          // L376: rule already in decision, no app session created
}
```

```
func extractUmData(umID string, eventSubs ..., threshold *models.UsageThreshold) (umData *models.UsageMonitoringData, err error) {
    if _, umExist := eventSubs[models.PcfPolicyAuthorizationAfEvent_USAGE_REPORT]; umExist {
        if threshold == nil {
            return nil, fmt.Errorf("usageThreshold is nil in USAGE REPORT Subscription")   // L1696
```

### The release is reached only through an app session that this path never created.

`HandleDeleteAppSessionContext` is a path to `RemovePccRule`, and it iterates the app session's own rule set:

```
for _, pccRuleID := range appSession.RelatedPccRuleIds {   // L473
    if err := smPolicy.RemovePccRule(pccRuleID, &deletedSmPolicyDec); err != nil {   // L474 -> delete(decision.PccRules, id)
```

`appSession.RelatedPccRuleIds` is populated only on the success path (L400), just before the app session is stored (`AppSessionPool.Store`, L433) and the function returns nil (L445). The failed request never reaches L400, so the rule it wrote at L269 is unreachable by any delete.

## To Reproduce

PCF v1.4.3, driven with NRF/UDR/PCF on loopback, `oauth: false`, a subscriber policy seeded in UDR, and one PDU session's SM policy created per UE. No change to the leaking code. Rules are counted by reading the stored decision through the PCF's own `GET /npcf-smpolicycontrol/v1/sm-policies/{smPolicyId}` endpoint, which returns `smPolicy.PolicyDecision`. The SM policy is created with a negotiated suppFeat of `0fff` (bit 12 set), i.e. an SMF that offers SponsoredConnectivity.

Each request is `POST /npcf-policyauthorization/v1/app-sessions` with one media component, `aspId`/`sponId` set, and a USAGE_REPORT event with no `usgThres` (the `extractUmData` trigger). This is the natural failure path, with no injection.

The leak: one field decides it

| case | app session created? | HTTP | rules held in the SM policy decision |
| --- | --- | --- | --- |
| A, normal create then delete | yes | 201, then 204 on delete | 1, then 0 after delete |
| B, fails at L376 (this bug) | no | 403, no app session | 1, not releasable |

Accumulation — same UE and SM policy, `afAppId` distinct per request:

| requests | rules held in the SM policy decision |
| --- | --- |
| 1 | 1 |
| 10 | 10 |
| 20 | 20 |
| 30 | 30 |

Slope exactly 1 per request, HTTP 403 REQUESTED_SERVICE_NOT_AUTHORIZED / "usageThreshold is nil in USAGE REPORT Subscription" every time, no ceiling. A QosData entry is added alongside each rule.

Control — same requests but a fixed `afAppId`: `GetPccRuleByAfAppId` reuses the existing rule, so the count stays at 1 across 10 requests. Growth needs a distinct `afAppId`, which the AF controls.

## Expected Behavior

When the sponsored-connectivity block takes an error exit (L376, and likewise the sibling exit at L383), the handler should roll back the rules it added to `smPolicy.PolicyDecision` in this call before returning, so a rejected app session leaves the decision as it found it.

## Screenshots

N/A

## Environment

- free5GC Version: PCF v1.4.3 (df6c0da); reproduced with a loopback NRF/UDR/PCF deployment
- OS: Ubuntu 24.04.5 LTS
- Kernel version: 6.8.0-139-generic
- go version: go1.25.5 linux/amd64

## Trace Files / Configuration Files

N/A. `oauth: false`; subscriber policy seeded in UDR; SM policy negotiated with suppFeat `0fff`.

## PCAP File

N/A. This is a code-logic issue, unrelated to the packet level.

## Log File

```
POST /npcf-policyauthorization/v1/app-sessions  (aspId/sponId set, USAGE_REPORT, no usgThres)
HTTP 403  {"cause":"REQUESTED_SERVICE_NOT_AUTHORIZED","detail":"usageThreshold is nil in USAGE REPORT Subscription"}
GET  /npcf-smpolicycontrol/v1/sm-policies/{id}  -> PolicyDecision.PccRules length = 1 (no app session owns it)
```

No app session is created, yet the SM policy decision retains one PCC rule. Over a 30-request run with distinct `afAppId`, the retained-rule count rises to 30.

## Additional Context

Beyond memory, the orphaned rules sit in `smPolicy.PolicyDecision`, which is what the handler sends to the SMF on the next successful app-session update (`policyauthorization.go:439-443`), so rules an AF was told it could not have (403) can still be provisioned toward the SMF/UPF on a later accepted request for the same PDU session. The entries are freed only when the SM policy is deleted (`HandleDeleteSmPolicyContextRequest`) — a PDU-session-scoped bound, which an AF can grow without limit within a live session by varying `afAppId`. The sibling exit at L383 (`handleSponsoredConnectivityInformation` error) has the identical defect.

## How did we find?

We found this bug by using our (work-in-progress) static analysis tool, manually verified it, and made a runnable PoC.

## See also

Root cause, fix process and exploit script: [REPORT.md](REPORT.md), [pcf_pcc_rule_leak_exploit.py](pcf_pcc_rule_leak_exploit.py). The SMF side of the propagation claim above is now measured: the SMF logs `In HandleSMPolicyUpdateNotify` + `apply sm policy decision error: too few fields 0` when the PCF hands it the tainted decision.
