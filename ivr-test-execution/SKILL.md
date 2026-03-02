---
name: ivr-test-execution
description: Execute and validate Amazon Connect IVR tests using the Chat API (StartChatContact). Generates a standalone Python script that runs all scenarios. Trigger on test IVR, run tests, execute tests, validate flow, verify IVR, run scenarios, or chat API test.
---

# IVR Test Execution Skill

Execute tests against deployed Amazon Connect IVR flows using the Chat API. Generate a standalone `run-tests.py` with all scenarios as data -- stable, reproducible, no AI at runtime.

For generating test scenario documents, use the `ivr-test-scripts` skill.

## Quick Reference

| Resource | Purpose |
|----------|---------|
| [execution-engine.md](execution-engine.md) | Script template, scenario data format, polling strategy, generation rules |
| [test-api-reference.md](test-api-reference.md) | Chat API + Participant API reference, IAM permissions, troubleshooting |

**Workflow:** Read test scripts + flow JSON -> extract scenarios -> generate `run-tests.py` -> user runs script -> read `test-results.json` -> generate `results-and-comparison.md`.

---

## How It Works

```
StartChatContact(flowId)
  -> Flow executes (same as Connect console "Test Chat")
  -> Play prompt blocks send text to chat
  -> ConnectParticipantWithLexBot waits for customer text
  -> Customer sends text via SendMessage ("provider", "1", etc.)
  -> Lex matches intent -> flow continues
  -> GetTranscript captures all system messages
  -> Compare actual vs expected
```

Text inputs work because Lex bots include DTMF digits and voice keywords as sample utterances. In chat mode, customer text goes directly to Lex for intent matching.

---

## Testable vs Not Testable

**Testable:** Lex bots (voice + DTMF), prompts, branching, Lambda invocations, contact attributes, queue transfers.

**Not testable -- check flow JSON before generating script, warn user, mark as SKIPPED:**

- **DTMF-only GetParticipantInput (no Lex)** -- takes Error branch in chat. Detect: `"Type": "GetParticipantInput"` without Lex bot.
- **StoreUserInput** -- voice-only, takes Error branch. Detect: `"Type": "StoreUserInput"`.
- **Audio-only prompts (no text fallback)** -- silent in chat transcript. Detect: `MessageParticipant` with only `PromptId`, no `Text`.
- **Voice session attributes** (barge-in, DTMF settings, audio timeouts) -- ignored in chat mode. Not a failure, but behavior differs from voice.

---

## Core Principles

1. **Stable script** -- generate `run-tests.py` with all scenarios as data. Runs independently, no AI at runtime.
2. **Chat API execution** -- `StartChatContact` -> `CreateParticipantConnection` -> `SendMessage` -> `GetTranscript` -> `DisconnectParticipant`.
3. **Deployed flows only** -- test flows already deployed to the Connect instance.
4. **Scan for untestable blocks** -- check flow JSON before generating. Warn user, mark affected scenarios SKIPPED.
5. **Test scripts as source of truth** -- scenarios map to IDs from `ivr-test-scripts` output (S1, S2, etc.).
6. **Read-only** -- never modify the flow to make a test pass.

---

## Prerequisite Questions

Before starting, collect all of these in a single prompt:

1. **AWS Profile** -- CLI profile name (e.g., `haohai`, `default`)
2. **Connect Instance** -- instance ID, or offer to run `aws connect list-instances`
3. **Contact Flow** -- flow ID, or offer to run `aws connect list-contact-flows`

Run `aws sso login --profile <PROFILE>` if needed. Confirm selections with user before proceeding.

---

## Execution Workflow

### Phase 1: Analyze

Read test scripts and flow JSON. Scan for untestable blocks (`GetParticipantInput` without Lex, `StoreUserInput`, audio-only prompts). Map each scenario to expect/send steps per the format in [execution-engine.md](execution-engine.md).

### Phase 2: Generate Script

Read [execution-engine.md](execution-engine.md). Generate `run-tests.py` with configuration, all scenarios as data, and the fixed execution engine. Do not modify the engine code per flow.

### Phase 3: Run

```bash
python3 run-tests.py                    # All scenarios
python3 run-tests.py --scenario S3      # Single scenario
python3 run-tests.py --tag happy-path   # By tag
python3 run-tests.py -v                 # Verbose
```

### Phase 4: Report

Read `test-results.json`. Generate `results-and-comparison.md` per the format below.

---

## Results Format

Produce `results-and-comparison.md`. One section per scenario with Expected (from test scripts) and Actual (from results). Mismatches marked with `>>> MISMATCH`.

```markdown
# [Flow Name] -- Results and Comparison

**Flow:** [flow-id] | **Method:** Chat API | **Date:** [date]

---

## S1: [Scenario Name]

**Expected:**
1. System plays greeting: "Welcome to..."
2. Customer sends: "provider"
3. System plays: "Thank you, provider..."

**Actual:**
1. Greeting contains "Welcome to" -- matched
2. Sent "provider"
3. Response contains "Thank you, provider" -- matched

**Result:** PASSED

---

## S17: [Scenario Name]

**Expected:**
1. System plays greeting
2. Customer sends: "I'd like to order a large pepperoni pizza"
3. System plays error: "Sorry, we did not understand..."

**Actual:**
1. Greeting received
2. Sent "I'd like to order a large pepperoni pizza"
3. >>> MISMATCH -- Expected "did not understand" but got: "We didn't catch that. Please try again."

**Result:** FAILED
```

**Rules:**

- Every scenario gets Expected and Actual sections
- Mark matches with checkmark, failures with `>>> MISMATCH`
- Result line: PASSED, FAILED, TIMEOUT, SKIPPED, or ERROR
- SKIPPED scenarios get reason (e.g., "Flow uses StoreUserInput -- untestable via Chat API")
- No summary tables or methodology sections -- just the scenario comparisons

---

## Cross-References

- [execution-engine.md](execution-engine.md) -- script template, scenario data format, generation rules, output checklist
- [test-api-reference.md](test-api-reference.md) -- Chat API operations, IAM permissions, troubleshooting
- `ivr-test-scripts` skill -- generates scenario documents before execution
- `amazon-connect-ivr` skill -- flow design rules affecting testability
