# Deployment Gotchas and Test API Reference

Reference for deployment bugs, AWS Native Test API behavior, and testing limitations discovered during real Amazon Connect deployments.

---

## Deployment Bugs

These cause `InvalidContactFlowException` on `create-contact-flow` with no useful error message.

### 1. Every block MUST have a `Transitions` key

Terminal blocks (`DisconnectParticipant`) need `"Transitions": {}`. Non-terminal blocks need `NextAction` plus an `Errors` array. `MessageParticipant` also needs `"Conditions": []`.

```json
// WRONG - missing Transitions entirely
{ "Identifier": "uuid", "Type": "DisconnectParticipant", "Parameters": {} }

// CORRECT
{ "Identifier": "uuid", "Type": "DisconnectParticipant", "Parameters": {}, "Transitions": {} }
```

### 2. `GetParticipantInput` MUST have its own `Text` parameter

The prompt text belongs inside `GetParticipantInput.Parameters.Text`. A preceding `MessageParticipant` greeting is fine, but the input block itself still needs `Text`.

### 3. Curly apostrophes break JSON parsing

Use only straight quotes/apostrophes in prompt text. Prefer "that is" over contractions.

### 4. `Loop` is not a valid action type

Use `UpdateContactAttributes` (counter) + `Compare` (check) + separate `GetParticipantInput` blocks. See [flow-components.md](flow-components.md) for the retry loop pattern.

### 5. Settings blocks with restricted error handling

These blocks reject `NoMatchingError` in Transitions -- use `"Errors": []` (empty array):
- `UpdateContactRecordingBehavior`
- `UpdateContactRoutingBehavior` (also needs `"Conditions": []`)

Exception: `UpdateContactEventHooks` DOES accept `NoMatchingError`.

### 6. `GetParticipantInput` with `StoreInput=True` rejects empty Conditions

When collecting free-form input (phone numbers, account numbers), omit the `Conditions` field entirely instead of passing `"Conditions": []`.

---

## AWS Native Test API Rules

Rules for writing testable IVR flows using `CreateTestCase` / `StartTestCaseExecution`.

### Rule 1: Never self-loop GetParticipantInput

A `GetParticipantInput` that loops back to itself does not emit a new `MessageReceived` event. Tests hang at `IN_PROGRESS`.

**Fix:** Route repeat/retry paths through an intermediate `MessageParticipant` block before re-entering the input block.

```
BAD:  GetParticipantInput → (repeat) → GetParticipantInput (same block)
GOOD: GetParticipantInput → MessageParticipant("One moment...") → GetParticipantInput
```

### Rule 2: Consecutive TTS blocks concatenate into one event

Events split only when a DTMF/input action occurs. Everything before input = one event. Everything after until next input = next event.

| Flow Sequence | Test Events |
|---|---|
| `Message("A") -> Input("B")` | 1 event: `"A. B"` |
| `Input("A") -> (DTMF) -> Message("B")` | 2 events: `"A"` then `"B"` |
| `Input("A") -> (DTMF) -> Message("B") -> Input("C")` | 2 events: `"A"` then `"B. C"` |

Use `MatchingCriteria: "Inclusion"` to match partial text across concatenated prompts.

### Rule 3: Retry counter pattern

Implement retries with `UpdateContactAttributes` + `Compare` + separate `GetParticipantInput` blocks (one per retry attempt). Each retry uses a different block, generating observable events.

### Rule 4: Error messages concatenate with the triggering prompt

When DTMF triggers `NoMatchingCondition`, the test engine prepends the `GetParticipantInput` prompt before the error message in a single `MessageReceived` event.

Example: Menu text "Press 1 for English" + error "Sorry, invalid selection" appears as: `"Press 1 for English. Sorry, invalid selection."`

### Rule 5: Silent blocks create event boundaries

Non-TTS blocks (`UpdateContactAttributes`, `Compare`) between TTS blocks cause the test engine to split events. A retry flow like `Input -> UpdateContactAttributes -> Message -> Input` produces two separate `MessageReceived` events.

### Rule 6: ConnectParticipantWithLexBot prompt is not observable

The `Text` parameter in `ConnectParticipantWithLexBot` does NOT appear as a `MessageReceived` event. Place an explicit `MessageParticipant` before the Lex block if you need the welcome message in test observations.

### Rule 7: CheckHoursOfOperation returns False without a queue

Always call `UpdateContactTargetQueue` before `CheckHoursOfOperation`. No queue = always closed.

### Rule 8: Lambda return values use `$.External.keyName`

`InvokeLambdaFunction` outputs are accessed via `$.External.keyName`. Max timeout: 8 seconds.

### Rule 9: Queue transfer requires two blocks

`TransferContactToQueue` cannot accept QueueId directly. Use `UpdateContactTargetQueue` (set ARN) then `TransferContactToQueue` (empty Parameters).

### Rule 10: TransferToFlow preserves contact context

TTS voice and contact attributes carry over to the sub-flow. Use full ARN for `ContactFlowId`.

### Rule 11: Generative TTS is Polly, not Nova Sonic

`TextToSpeechEngine: "Generative"` uses Amazon Polly Generative (higher-quality TTS). Nova Sonic is a separate Bedrock model requiring console-only setup. Verified voice: `Matthew`. See [flow-components.md](flow-components.md) for the full comparison table.

### Rule 12: CreateWisdomSession is a silent block

No `MessageReceived` event is emitted. Place a `MessageParticipant` after it to confirm success in tests. Requires QIC assistant association via `aws connect create-integration-association --integration-type WISDOM_ASSISTANT`.

### Rule 13: Test API rejects empty Text in observations

Every `MessageReceived` observation must have non-empty `Text`. Use `MatchingCriteria: "Inclusion"` with a keyword fragment instead of empty string.

### Rule 14: Multi-turn QIC conversations need self-loop

For Q in Connect flows, `ConnectParticipantWithLexBot` MUST loop back to itself:
- `NextAction` points to the same block
- `NoMatchingError` also points to the same block
- `EndConversationIntent` condition routes to disconnect

Without the loop, QIC disconnects after the first AI response.

```json
{
  "Type": "ConnectParticipantWithLexBot",
  "Transitions": {
    "NextAction": "self-block-id",
    "Conditions": [
      { "NextAction": "disconnect-id", "Condition": { "Operator": "Equals", "Operands": ["EndConversationIntent"] } }
    ],
    "Errors": [
      { "NextAction": "self-block-id", "ErrorType": "NoMatchingCondition" }
    ]
  }
}
```

---

## Testing Limitations

### Chat API does not surface MessageParticipant blocks

Flows designed for voice may not emit `MessageParticipant` as chat messages when tested via `StartChatContact` / `SendMessage` / `GetTranscript`. Only customer messages appear in transcripts.

**Workaround:** Test via Connect Console "Test Chat"/"Test Voice" features, or via real phone calls. The flows work correctly through these channels.
