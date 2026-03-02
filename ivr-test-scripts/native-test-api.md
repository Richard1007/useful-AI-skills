# AWS Native Test API Reference

Reference for creating programmatic tests using `CreateTestCase` / `StartTestCaseExecution`. Manual test scripts follow different rules.

## Test Content Schema

Version MUST be `2019-10-30`.

```json
{
  "Version": "2019-10-30",
  "Metadata": {},
  "Observations": [
    {
      "Identifier": "unique-uuid",
      "Event": {
        "Identifier": "event-uuid",
        "Type": "TestInitiated | MessageReceived | FlowActionStarted | TestCompleted",
        "Actor": "System",
        "Properties": { "Text": "partial text to match" },
        "MatchingCriteria": "Inclusion"
      },
      "Actions": [
        {
          "Identifier": "action-uuid",
          "Type": "SendInstruction | TestControl",
          "Actor": "Customer",
          "Parameters": {
            "DtmfInput": { "Type": "DtmfInput", "Properties": { "Value": 1 } }
          }
        }
      ],
      "Transitions": { "NextObservations": ["next-obs-uuid"] }
    }
  ]
}
```

## Validation Rules

1. Version MUST be `"2019-10-30"` — no other version accepted
2. Do NOT include `Usage` field — rejected by validation
3. Do NOT include `Transitions` inside Actions — only Observations have Transitions
4. DTMF Value must be a number — `{ "Value": 1 }` not `{ "Value": "1" }`
5. `FlowActionStarted` events CANNOT have Actions — use `MessageReceived` for caller actions
6. Use `Status: 'PUBLISHED'` for validation — `SAVED` accepts anything but won't execute
7. `Content` parameter is a JSON string — stringify the test JSON before passing to `CreateTestCase`
8. Every `MessageReceived` observation MUST have non-empty `Text` — empty text throws `InvalidTestCaseException`

## Message Concatenation

Consecutive TTS-producing blocks are concatenated into a SINGLE `MessageReceived` event. Events split when a DTMF/input action occurs.

| Flow Sequence | Test API Events |
|--------------|-----------------|
| `MessageParticipant("A") -> GetParticipantInput("B")` | ONE event: `"A. B"` |
| `MessageParticipant("A") -> MessageParticipant("B") -> GetParticipantInput("C")` | ONE event: `"A. B. C"` |
| `GetParticipantInput("A")` -> DTMF -> `MessageParticipant("B")` | TWO events: `"A"` then `"B"` |
| `GetParticipantInput("A")` -> DTMF -> `MessageParticipant("B") -> GetParticipantInput("C")` | TWO events: `"A"` then `"B. C"` |

**Rule:** Everything before input = one event. Everything after input until next input = next event. Use `MatchingCriteria: "Inclusion"` to match partial text within concatenated messages.

## Observation Structure

```json
{
  "Identifier": "obs-1",
  "Event": {
    "Identifier": "evt-1",
    "Type": "TestInitiated",
    "Actor": "System"
  },
  "Actions": [],
  "Transitions": { "NextObservations": ["obs-2"] }
},
{
  "Identifier": "obs-2",
  "Event": {
    "Identifier": "evt-2",
    "Type": "MessageReceived",
    "Actor": "System",
    "Properties": { "Text": "Welcome to our service" },
    "MatchingCriteria": "Inclusion"
  },
  "Actions": [
    {
      "Identifier": "act-1",
      "Type": "SendInstruction",
      "Actor": "Customer",
      "Parameters": {
        "DtmfInput": { "Type": "DtmfInput", "Properties": { "Value": 1 } }
      }
    }
  ],
  "Transitions": { "NextObservations": ["obs-3"] }
}
```

## Event Behavior by Block Type

| Block Type | Observable? | Event Behavior | Notes |
|-----------|-------------|----------------|-------|
| `MessageParticipant` | Yes | Produces `MessageReceived` | Lambda-interpolated `$.External.*` values are resolved in the event text. |
| `GetParticipantInput` | Yes | Produces `MessageReceived` | Self-loop back to same block is **untestable** — test hangs at `IN_PROGRESS`. |
| `ConnectParticipantWithLexBot` | Partial | DTMF works; prompt text NOT observable | Match the `MessageParticipant` before Lex, then the response after intent routing. |
| `TransferToFlow` | Yes | Continues observation across flows | Sub-flow `MessageParticipant` events are observable. Enables end-to-end multi-flow testing. |
| `TransferContactToQueue` | No | No event produced | End observations after pre-transfer message. Use `TestCompleted` after final message. |
| `UpdateContactAttributes` | No | Silent | Standard concatenation boundary rules apply. |
| `Compare` | No | Silent | Standard concatenation boundary rules apply. |
| `UpdateContactTargetQueue` | No | Silent | |
| `CheckHoursOfOperation` | No | Silent, but evaluates real-time | Uses actual queue hours during execution. Use 24/7 queue for "open" path, restricted queue for "closed" path. |
| `InvokeLambdaFunction` | No | Silent | Lambda executes; return values available for interpolation in subsequent blocks. |
| `UpdateContactRecordingBehavior` | No | Silent | |
| `DistributeByPercentage` | No | Silent, non-deterministic | Cannot write deterministic tests. Temporarily set 100% to target branch. |
| `UpdateContactRoutingBehavior` | No | Silent | Priority changes happen silently. |
| `UpdateContactEventHooks` | No | Silent | Event hook assignments happen silently. |
| `CreateWisdomSession` | No | Silent | Q Connect session created silently. Creates event boundary like other silent blocks. |

**Case normalization:** Neural TTS normalizes case in output. Use `Inclusion` matching with lowercase-safe text fragments.

**TTS engines:** Generative TTS (Polly Generative, NOT Nova Sonic) produces identical `MessageReceived` events to Neural/Standard. No special handling needed.

**Mismatched branches:** If the flow takes an unexpected branch, mismatched text gets absorbed into the preceding observation's actual text. Verify flow logic matches test expectations.

## Error Path Events

When `GetParticipantInput` triggers `NoMatchingCondition` and the path goes through silent blocks before reaching a new TTS block, expect MULTIPLE separate `MessageReceived` events.

**Pattern:**
```
GetParticipantInput("Press 1 for English")
  -> invalid DTMF
  -> UpdateContactAttributes(retryCount++)  [silent]
  -> Compare(retryCount < 3)  [silent]
  -> MessageParticipant("Sorry, invalid selection")
  -> GetParticipantInput("Please try again. Press 1 for English")
```

**Correct observation pattern for retry flows:**
```json
{
  "Identifier": "obs-welcome-menu",
  "Event": { "Type": "MessageReceived", "Properties": { "Text": "Press 1 for English" } },
  "Actions": [{ "Type": "SendInstruction", "Parameters": { "DtmfInput": { "Value": 9 } } }],
  "Transitions": { "NextObservations": ["obs-error-msg"] }
},
{
  "Identifier": "obs-error-msg",
  "Event": { "Type": "MessageReceived", "Properties": { "Text": "Sorry, invalid selection" }, "MatchingCriteria": "Inclusion" },
  "Actions": [],
  "Transitions": { "NextObservations": ["obs-retry-menu"] }
},
{
  "Identifier": "obs-retry-menu",
  "Event": { "Type": "MessageReceived", "Properties": { "Text": "Please try again" }, "MatchingCriteria": "Inclusion" },
  "Actions": [{ "Type": "SendInstruction", "Parameters": { "DtmfInput": { "Value": 1 } } }],
  "Transitions": { "NextObservations": ["obs-result"] }
}
```

## Test Execution Commands

```bash
# Create test case (Content is stringified JSON)
aws connect create-test-case \
  --instance-id <INSTANCE_ID> \
  --contact-flow-id <FLOW_ID> \
  --name "S1: Happy Path - Sales" \
  --description "Press 1 to reach sales queue" \
  --status PUBLISHED \
  --content "$(cat test-case.json | jq -c .)" \
  --profile <PROFILE>

# Start execution
aws connect start-test-case-execution \
  --instance-id <INSTANCE_ID> \
  --test-case-id <TEST_CASE_ID> \
  --profile <PROFILE>

# Poll execution status
aws connect get-test-case-execution \
  --instance-id <INSTANCE_ID> \
  --test-case-execution-id <EXECUTION_ID> \
  --profile <PROFILE> \
  --query 'TestCaseExecution.Status'

# Get detailed results
aws connect get-test-case-execution \
  --instance-id <INSTANCE_ID> \
  --test-case-execution-id <EXECUTION_ID> \
  --profile <PROFILE>
```

## Known Limitations

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| Self-loop `GetParticipantInput` | Test hangs at `IN_PROGRESS`; no new event for re-played prompt | Route repeat paths through intermediate `MessageParticipant`. See `amazon-connect-ivr` skill. Document as "Manual testing only." |
| `DistributeByPercentage` | Non-deterministic routing | Temporarily set 100% to target branch, test, restore original percentages. |
| `TransferContactToQueue` | No observable event | End observations after pre-transfer message. |
| Case normalization | Neural TTS may change case | Use `Inclusion` matching with lowercase-safe fragments. |
| Empty observation text | Rejected with `InvalidTestCaseException` | Every `MessageReceived` must have non-empty `Text`. Use keyword fragment with `Inclusion`. |
