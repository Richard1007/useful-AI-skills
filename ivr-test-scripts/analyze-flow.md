# Analyze IVR Flow

## Step 1: Parse the Flow JSON

Load the flow JSON and build a graph:

```python
import json

with open("flow.json") as f:
    flow = json.load(f)

actions = {a["Identifier"]: a for a in flow["Actions"]}
start = flow["StartAction"]
```

## Step 2: Walk All Paths

Starting from `StartAction`, perform a depth-first traversal. At each node:

1. **Record the block type and parameters** (especially `Text` for prompts)
2. **At branch points** (GetParticipantInput, ConnectParticipantWithLexBot, Compare):
   - Record each condition (DTMF digit or intent name)
   - Record error branches (NoMatchingCondition, NoMatchingError, InputTimeLimitExceeded)
   - Record default/fallthrough path
3. **At terminal blocks** (DisconnectParticipant): Record as path end

## Step 3: Build the Path Tree

Represent the IVR as a tree:

```
Root (StartAction)
├── [Block: Set Voice] → auto-advance
├── [Block: Greeting] → auto-advance
├── [Block: Main Menu - Lex Bot]
│   ├── Condition: ProviderIntent (DTMF "1" / Voice "provider")
│   │   ├── [Block: Provider Prompt] → auto-advance
│   │   └── [Block: Provider Menu - Lex Bot]
│   │       ├── Condition: ReferralIntent (DTMF "1" / Voice "referral")
│   │       │   └── [Block: Referral Prompt] → Disconnect
│   │       ├── Condition: PriorAuthIntent (DTMF "2" / Voice "prior auth")
│   │       │   └── [Block: Prior Auth Prompt] → Disconnect
│   │       ├── Condition: ClaimsBillingIntent (DTMF "3" / Voice "claims")
│   │       │   └── [Block: Claims Prompt] → Disconnect
│   │       ├── Error: NoMatchingCondition → Error Prompt → Disconnect
│   │       └── Error: NoMatchingError → Error Prompt → Disconnect
│   ├── Condition: PatientIntent (DTMF "2" / Voice "patient")
│   │   ├── [Block: Patient Prompt] → auto-advance
│   │   └── [Block: Patient Menu - Lex Bot]
│   │       ├── Condition: AppointmentIntent (DTMF "1" / Voice "appointment")
│   │       │   └── [Block: Appointment Prompt] → Disconnect
│   │       ├── Condition: PrescriptionIntent (DTMF "2" / Voice "prescription")
│   │       │   └── [Block: Prescription Prompt] → Disconnect
│   │       ├── Condition: TestResultsIntent (DTMF "3" / Voice "test results")
│   │       │   └── [Block: Test Results Prompt] → Disconnect
│   │       ├── Error: NoMatchingCondition → Error Prompt → Disconnect
│   │       └── Error: NoMatchingError → Error Prompt → Disconnect
│   ├── Error: NoMatchingCondition → Error Prompt → Disconnect
│   └── Error: NoMatchingError → Error Prompt → Disconnect
```

## Step 4: Extract Prompt Text

For each `MessageParticipant` and `ConnectParticipantWithLexBot` block, extract the exact `Parameters.Text` value. This is what the system says to the caller.

## Step 5: Map DTMF and Voice Inputs

For `ConnectParticipantWithLexBot` blocks:
- Each `Condition.Operands[0]` is an intent name
- Map intent name → DTMF digit (from Lex bot utterances)
- Map intent name → voice phrases (from Lex bot utterances)

For `GetParticipantInput` blocks:
- Each `Condition.Operands[0]` is the DTMF digit directly

## Step 6: Enumerate All Unique Paths

List every unique path from start to disconnect:

```
Path 1: Greeting → Main Menu [Provider/1] → Provider Prompt → Provider Menu [Referral/1] → Referral Prompt → Disconnect
Path 2: Greeting → Main Menu [Provider/1] → Provider Prompt → Provider Menu [Prior Auth/2] → Prior Auth Prompt → Disconnect
Path 3: Greeting → Main Menu [Provider/1] → Provider Prompt → Provider Menu [Claims/3] → Claims Prompt → Disconnect
Path 4: Greeting → Main Menu [Patient/2] → Patient Prompt → Patient Menu [Appointment/1] → Appointment Prompt → Disconnect
Path 5: Greeting → Main Menu [Patient/2] → Patient Prompt → Patient Menu [Prescription/2] → Prescription Prompt → Disconnect
Path 6: Greeting → Main Menu [Patient/2] → Patient Prompt → Patient Menu [Test Results/3] → Test Results Prompt → Disconnect
Path 7: Greeting → Main Menu [Error] → Error Prompt → Disconnect
Path 8: Greeting → Main Menu [Provider/1] → Provider Prompt → Provider Menu [Error] → Error Prompt → Disconnect
Path 9: Greeting → Main Menu [Patient/2] → Patient Prompt → Patient Menu [Error] → Error Prompt → Disconnect
```

Then multiply by input method (DTMF vs Voice) for Lex-integrated menus.
