# Issue #1185 — [BUG] PCF: app-session modify leaves an orphaned PCC rule in the SM policy decision when extractUmData fails

- Repository: free5gc/free5gc
- URL: https://github.com/free5gc/free5gc/issues/1185
- Author: ae878
- Opened: Oct 3, 2026
- State: Open

## Bug Description

When an AF updates an application session, the PCF's `HandleModAppSessionContext` processes the media components and, for a media sub-component carrying a new flow, creates a PCC rule and installs it into the stored SM policy decision (`smPolicy.PolicyDecision.PccRules`). The new rule id is placed only in a local map that is written back to `appSession.RelatedPccRuleIds` at the very end of the handler. Two sponsored-connectivity failure exits sit between the install and that write-back; on either exit the handler returns without removing the rule and without registering it on the app session.

The paired release, `(*UeSmPolicyData).RemovePccRule`, is reached through `HandleDeleteAppSessionContext`, which walks `appSession.RelatedPccRuleIds`. Because the failed modify never wrote the id there, a later delete of the app session cannot free the rule; it is stranded in the SM policy decision for the life of the PDU session.

This is the modify-handler analogue of the same defect in `postAppSessCtxProcedure` (the app-session create path). The failure exit is taken whenever an AF asks for a USAGE_REPORT event subscription without a `usgThres`.

Source analysis on PCF v1.4.3, commit df6c0da. Still present on upstream main (eb57557, 2026-08-20): `HandleModAppSessionContext` (`policyauthorization.go:544`) still installs the rule into the decision (`:659`) and still writes `appSession.RelatedPccRuleIds` only at the end (`:804`), after the `extractUmData` exit (`:785`) — so the gap is unchanged on main. Measured live on the stock `free5gc/pcf:v4.3.0` image (which is built from eb57557): each rejected modify left one rule that survived the subsequent app-session delete, 5 modifies -> 5 stranded rules.

### The rule is installed into the stored decision first.

`handleMediaSubComponent` (called at L606) creates the rule via `util.CreatePccRule` (L81) and installs it into the stored decision with `SetPccRuleRelatedData` (L104) and a direct write (L122). Back in the caller, the new rule id goes only into a local `relatedPccRuleIds` map.

```
// inside handleMediaSubComponent
pccRule = util.CreatePccRule(smPolicy.PccRuleIdGenerator, precedence, nil, "")             // L81
...
util.SetPccRuleRelatedData(smPolicy.PolicyDecision, pccRule, tcData, &qosData, nil, nil)   // L104
...
smPolicy.PolicyDecision.PccRules[pccRule.PccRuleId] = pccRule                               // L122: decision.PccRules[id] = rule
```

### The failure exit returns before the id is registered on the app session.

`extractUmData` returns an error for a USAGE_REPORT subscription with no `usgThres`, and the handler returns:

```
if tempUmData, err := extractUmData(umID, eventSubs, ascUpdateData.EvSubsc.UsgThres); err != nil {
    problemDetail := util.GetProblemDetail(err.Error(), util.REQUESTED_SERVICE_NOT_AUTHORIZED)
    ...
    return nil, &problemDetail       // L776-782: rule already in decision; RelatedPccRuleIds not written
}
```

The write-back that would make the rule releasable happens only after both failure exits:

```
appSession.RelatedPccRuleIds = relatedPccRuleIds   // L794-795: reached only on success
```

### The release walks RelatedPccRuleIds, which the failed modify never updated.

```
for _, pccRuleID := range appSession.RelatedPccRuleIds {   // L473
    if err := smPolicy.RemovePccRule(pccRuleID, &deletedSmPolicyDec); err != nil {   // L474
```

The id of the rule left by the failed modify is not in `RelatedPccRuleIds`, so `HandleDeleteAppSessionContext` never frees it.

## To Reproduce

PCF v1.4.3, verified with a deterministic Go unit test on the real `(*Processor).HandleModAppSessionContext`, no fault injection (both failure branches reached with plain realistic AF PATCH bodies). Metric = `len(smPolicy.PolicyDecision.PccRules)` before/after the failed modify and after a subsequent `HandleDeleteAppSessionContext`.

Setup: one `AppSessionData`; a `SmPolicyData` with an empty `PolicyDecision.PccRules`, SuppFeat "0800" (bit 12 set, bit 5 clear), and empty `RelatedPccRuleIds`. Each PATCH adds one media sub-component (fDescs ["permit out ip from ... to any"]), `aspId`/`sponId` set, a USAGE_REPORT event, and no `usgThres`.

The leak

| step | PccRules in SM policy decision |
| --- | --- |
| before failed modify | 0 |
| after failed modify (HTTP 403) | 1 |
| after HandleDeleteAppSessionContext | 1 (rule survives delete) |
| control: a properly owned rule, after delete | 0 |

Accumulation (same UE / app session / SM policy, distinct flow per request):

| failed modifies | PccRules in decision |
| --- | --- |
| 1 | 1 |
| 5 | 5 |

Same flow reused → bounded at 1 (dedup by flow description). A QosData + TrafficControlData leaks alongside each rule.

## Expected Behavior

On the error exits (L776, and the sibling sponsored-connectivity exit at L785), the handler should roll back the rules it added to `smPolicy.PolicyDecision` in this call before returning, so a rejected modify leaves the decision as it found it.

## Screenshots

N/A

## Environment

- free5GC Version: PCF v1.4.3 (df6c0da); reproduced through the PCF processor unit-test harness
- OS: Ubuntu 24.04.5 LTS
- Kernel version: 6.8.0-139-generic
- go version: go1.25.5 linux/amd64

## Trace Files / Configuration Files

N/A. Test-only; no change to product code.

## PCAP File

N/A. This is a code-logic issue, unrelated to the packet level.

## Log File

```
[11 path35] HTTP=403 body={"status":403,"detail":"usageThreshold is nil in USAGE REPORT Subscription","cause":"REQUESTED_SERVICE_NOT_AUTHORIZED"}
[11 path35] PccRules on UeSmPolicyData: before=0 after-failed-modify=1
[11 path35] appSession.RelatedPccRuleIds=map[] (new rule NOT registered)
[11 path35] after HandleDeleteAppSessionContext: PccRules=1 (LEAK: rule survives delete)
```

## Additional Context

The orphaned rule sits in `smPolicy.PolicyDecision`, which the PCF sends to the SMF on the next successful app-session update for the same PDU session, so a rule an AF was rejected for (403) can still be provisioned toward the SMF/UPF on a later accepted request. The sibling exit at L785 (`handleSponsoredConnectivityInformation`, UMC not supported) has the identical defect. Rules are freed only when the SM policy (PDU session) is deleted.

Root cause, fix process and exploit script: [REPORT.md](REPORT.md), [pcf_pcc_rule_leak_exploit.py](pcf_pcc_rule_leak_exploit.py).

## How did we find?

We found this bug by using our (work-in-progress) static analysis tool, manually verified it, and made a runnable PoC.
