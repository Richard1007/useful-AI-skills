# QA and Validation

**Assume there are problems. Your job is to find them.**

Your first generated flow is almost never perfect. Approach QA as a bug hunt, not a confirmation step.

## Phase 1: Structural Validation

Run these checks on the JSON:

### 1. JSON Validity
- Parse the JSON — must be valid JSON with no syntax errors
- Must have `Version`, `StartAction`, `Metadata`, and `Actions` top-level keys
- `Version` must be `"2019-10-30"`

### 2. UUID Integrity
- [ ] Every `Identifier` in Actions is a valid UUID v4 format
- [ ] No duplicate `Identifier` values
- [ ] `StartAction` references a valid Identifier
- [ ] Every `NextAction` in Transitions references a valid Identifier
- [ ] Every `NextAction` in Conditions references a valid Identifier
- [ ] Every `NextAction` in Errors references a valid Identifier

### 3. Reachability
- [ ] Every block is reachable from `StartAction` (no orphaned blocks)
- [ ] Every path from `StartAction` eventually reaches a `DisconnectParticipant` or transfer block

### 4. Error Handling
- [ ] Every `GetParticipantInput` has `InputTimeLimitExceeded`, `NoMatchingCondition`, and `NoMatchingError` error handlers
- [ ] Every `ConnectParticipantWithLexBot` has `NoMatchingCondition` and `NoMatchingError` error handlers
- [ ] Every `MessageParticipant` with potential errors has `NoMatchingError` handler
- [ ] Error handlers route to meaningful blocks (not dead ends)

### 5. Metadata
- [ ] Every action has an entry in `ActionMetadata` with a `position`
- [ ] Positions don't overlap (blocks should be visually separated)
- [ ] `entryPointPosition` is set

## Phase 2: Content Validation

### 1. Prompt Text
- [ ] No placeholder text remaining (XXXX, TODO, TBD, Lorem Ipsum)
- [ ] Prompts are grammatically correct and natural-sounding
- [ ] DTMF menu prompts list all available options
- [ ] Error prompts are helpful and informative

### 2. DTMF Conditions
- [ ] Each DTMF option in the prompt has a matching Condition
- [ ] No conditions reference options not mentioned in the prompt
- [ ] `InputTimeLimitSeconds` is reasonable (3-10 seconds typically)

### 3. Lex Integration (if used)
- [ ] `LexV2Bot.AliasArn` is a valid ARN format
- [ ] All intent names in Conditions match actual intents in the Lex bot
- [ ] The prompt text guides the caller on what to say

## Phase 3: Deployment Validation

### 1. Dry Run Upload
```bash
aws connect create-contact-flow \
  --instance-id <INSTANCE_ID> \
  --name "TEST-<FlowName>" \
  --type CONTACT_FLOW \
  --content "$(cat flow.json)" \
  --profile <PROFILE>
```

If it fails, the error message will tell you what's wrong. Common issues:
- Invalid action type
- Missing required parameters
- Invalid transition references
- Malformed ARN

### 2. Post-Upload Verification
```bash
# Fetch the flow back and compare
aws connect describe-contact-flow \
  --instance-id <INSTANCE_ID> \
  --contact-flow-id <FLOW_ID> \
  --profile <PROFILE>
```

## Verification Loop

1. Generate/edit the flow JSON
2. Run Phase 1 structural checks
3. Run Phase 2 content checks
4. **List all issues found** (if none found, look again more critically)
5. Fix all issues
6. **Re-verify** — one fix often creates another problem
7. Attempt deployment (Phase 3)
8. If deployment fails, diagnose error, fix, go to step 2
9. Repeat until a full pass reveals no new issues AND deployment succeeds

**Do not declare success until:**
- All Phase 1 and 2 checks pass
- The flow deploys without errors
- You have completed at least one fix-and-verify cycle

## Quick Validation Script

Use this Python snippet to validate structural integrity:

```python
import json, sys

def validate_flow(flow):
    issues = []
    actions = {a["Identifier"]: a for a in flow["Actions"]}

    # Check StartAction exists
    if flow["StartAction"] not in actions:
        issues.append(f"StartAction {flow['StartAction']} not found in Actions")

    # Check all transition targets exist
    for action in flow["Actions"]:
        t = action.get("Transitions", {})
        if "NextAction" in t and t["NextAction"] and t["NextAction"] not in actions:
            issues.append(f"Block {action['Identifier']}: NextAction {t['NextAction']} not found")
        for cond in t.get("Conditions", []):
            if cond["NextAction"] not in actions:
                issues.append(f"Block {action['Identifier']}: Condition target {cond['NextAction']} not found")
        for err in t.get("Errors", []):
            if err["NextAction"] not in actions:
                issues.append(f"Block {action['Identifier']}: Error target {err['NextAction']} not found")

    # Check for duplicate IDs
    ids = [a["Identifier"] for a in flow["Actions"]]
    if len(ids) != len(set(ids)):
        issues.append("Duplicate Identifiers found")

    # Check reachability
    reachable = set()
    def walk(uid):
        if uid in reachable or uid not in actions:
            return
        reachable.add(uid)
        t = actions[uid].get("Transitions", {})
        if "NextAction" in t and t["NextAction"]:
            walk(t["NextAction"])
        for c in t.get("Conditions", []):
            walk(c["NextAction"])
        for e in t.get("Errors", []):
            walk(e["NextAction"])
    walk(flow["StartAction"])
    orphans = set(actions.keys()) - reachable
    if orphans:
        issues.append(f"Orphaned blocks: {orphans}")

    # Check all paths terminate
    for uid in reachable:
        a = actions[uid]
        if a["Type"] == "DisconnectParticipant":
            continue
        t = a.get("Transitions", {})
        if not t:
            if a["Type"] != "DisconnectParticipant":
                issues.append(f"Block {uid} ({a['Type']}) has no transitions and is not a disconnect")

    return issues

with open(sys.argv[1]) as f:
    flow = json.load(f)
issues = validate_flow(flow)
if issues:
    print("ISSUES FOUND:")
    for i in issues:
        print(f"  - {i}")
else:
    print("ALL CHECKS PASSED")
```

Save as `validate_flow.py` and run: `python3 validate_flow.py flow.json`
