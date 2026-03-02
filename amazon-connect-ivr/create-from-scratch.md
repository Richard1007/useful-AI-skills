# Create IVR From Scratch

## Workflow

### Step 1: Gather Requirements

Before generating any JSON, confirm these with the user:

1. **Flow name** — What should this IVR be called?
2. **Greeting** — What should the first prompt say?
3. **Menu structure** — What options does the caller have? How many levels?
4. **Input method** — DTMF only, voice only, both (DTMF + voice via Lex), or Nova Sonic Speech-to-Speech?
5. **Error handling** — What to say on invalid input? How many retries?
6. **Termination** — How does the flow end? (Disconnect, transfer to queue, transfer to agent?)
7. **TTS voice** — Which voice? (Default: Matthew, Generative engine)

### Step 2: Design the Flow Tree

Draw out the flow tree before writing JSON:

```
Entry
  → Set Voice (optional)
  → Greeting Prompt
  → Menu (GetParticipantInput or ConnectParticipantWithLexBot)
    ├→ Option 1 → Sub-menu or Prompt → End
    ├→ Option 2 → Sub-menu or Prompt → End
    ├→ Option N → Sub-menu or Prompt → End
    ├→ Error/NoMatch → Error Prompt → End or Retry
    └→ Timeout → Timeout Prompt → End or Retry
```

### Step 3: Generate UUIDs

Generate a unique UUID v4 for EVERY block. Use Python:

```python
import uuid
str(uuid.uuid4())
```

Or use the pattern: `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx` where x is hex and y is one of 8,9,a,b.

### Step 4: Build the JSON

Follow this order:
1. Create the `DisconnectParticipant` block(s) first — these are the terminal nodes
2. Create `MessageParticipant` blocks for each prompt (greeting, options, errors)
3. Create the menu block (`GetParticipantInput` or `ConnectParticipantWithLexBot`)
4. Wire transitions from menu conditions to the appropriate prompts
5. Wire error transitions
6. Set the `StartAction` to the first block
7. Build `Metadata` with positions for each block

### Step 5: Deciding Between DTMF, Lex, and Nova Sonic

**Use DTMF only (`GetParticipantInput`)** when:
- Options are simple numbered choices (Press 1, 2, 3...)
- No voice recognition needed
- Simpler to implement, no Lex bot dependency

**Use Lex V2 (`ConnectParticipantWithLexBot`)** when:
- User wants to "say or press" options
- Natural language understanding is needed
- Multiple ways to express the same intent (e.g., "provider", "doctor", "I'm a physician")

When using Lex:
1. First create the Lex bot using the `amazon-lex-bot` skill
2. Get the bot alias ARN
3. Use `ConnectParticipantWithLexBot` with the alias ARN
4. Map intent names in `Conditions` (not DTMF digits)

**Use Nova Sonic Speech-to-Speech (`ConnectParticipantWithLexBot` + Generative engine)** when:
- Low-latency, human-like conversational AI is needed
- The IVR should handle open-ended natural language (not just intent matching)
- Expressive, adaptive voice responses are desired
- Barge-in / interruption handling must be seamless
- The Connect instance is in **us-east-1** or **us-west-2**

When using Nova Sonic:
1. Create a Conversational AI bot in the Amazon Connect console
2. Set the bot locale's speech model to **Speech-to-Speech: Amazon Nova Sonic**
3. Build the bot language
4. Use `UpdateContactTextToSpeechVoice` with `TextToSpeechEngine: "Generative"` and a Nova Sonic voice
5. Use `ConnectParticipantWithLexBot` with the S2S-enabled bot alias ARN
6. See the **Nova Sonic Speech-to-Speech IVR** template below

### Step 6: Sub-menus

For multi-level IVRs, nest menus by chaining:
```
Main Menu → Option 1 → Sub-Menu 1 → Option 1a → Prompt → Disconnect
                                    → Option 1b → Prompt → Disconnect
          → Option 2 → Sub-Menu 2 → ...
```

Each sub-menu is another `GetParticipantInput` or `ConnectParticipantWithLexBot` block.

## Template: Simple DTMF Menu

```json
{
  "Version": "2019-10-30",
  "StartAction": "SET_VOICE_UUID",
  "Metadata": {
    "entryPointPosition": { "x": 39, "y": 40 },
    "ActionMetadata": {
      "SET_VOICE_UUID": { "position": { "x": 250, "y": 200 } },
      "GREETING_UUID": { "position": { "x": 550, "y": 200 } },
      "MENU_UUID": { "position": { "x": 850, "y": 200 } },
      "OPT1_UUID": { "position": { "x": 1200, "y": 50 } },
      "OPT2_UUID": { "position": { "x": 1200, "y": 250 } },
      "ERROR_UUID": { "position": { "x": 1200, "y": 450 } },
      "DISCONNECT_UUID": { "position": { "x": 1550, "y": 250 } }
    }
  },
  "Actions": [
    {
      "Identifier": "SET_VOICE_UUID",
      "Type": "UpdateContactTextToSpeechVoice",
      "Parameters": {
        "TextToSpeechVoice": "Matthew",
        "TextToSpeechEngine": "Generative",
        "TextToSpeechStyle": "None"
      },
      "Transitions": {
        "NextAction": "GREETING_UUID",
        "Errors": [
          { "NextAction": "GREETING_UUID", "ErrorType": "NoMatchingError" }
        ]
      }
    },
    {
      "Identifier": "GREETING_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "Welcome to ..." },
      "Transitions": { "NextAction": "MENU_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "MENU_UUID",
      "Type": "GetParticipantInput",
      "Parameters": {
        "Text": "Press 1 for ... Press 2 for ...",
        "StoreInput": "False",
        "InputTimeLimitSeconds": "5"
      },
      "Transitions": {
        "NextAction": "ERROR_UUID",
        "Conditions": [
          { "NextAction": "OPT1_UUID", "Condition": { "Operator": "Equals", "Operands": ["1"] } },
          { "NextAction": "OPT2_UUID", "Condition": { "Operator": "Equals", "Operands": ["2"] } }
        ],
        "Errors": [
          { "NextAction": "ERROR_UUID", "ErrorType": "InputTimeLimitExceeded" },
          { "NextAction": "ERROR_UUID", "ErrorType": "NoMatchingCondition" },
          { "NextAction": "ERROR_UUID", "ErrorType": "NoMatchingError" }
        ]
      }
    },
    {
      "Identifier": "OPT1_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "You selected option 1." },
      "Transitions": { "NextAction": "DISCONNECT_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "OPT2_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "You selected option 2." },
      "Transitions": { "NextAction": "DISCONNECT_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "ERROR_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "Sorry, we didn't understand your selection. Goodbye." },
      "Transitions": { "NextAction": "DISCONNECT_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "DISCONNECT_UUID",
      "Type": "DisconnectParticipant",
      "Parameters": {},
      "Transitions": {}
    }
  ]
}
```

## Template: Lex V2 Menu (Voice + DTMF)

```json
{
  "Version": "2019-10-30",
  "StartAction": "SET_VOICE_UUID",
  "Metadata": { "..." : "..." },
  "Actions": [
    {
      "Identifier": "SET_VOICE_UUID",
      "Type": "UpdateContactTextToSpeechVoice",
      "Parameters": {
        "TextToSpeechVoice": "Matthew",
        "TextToSpeechEngine": "Generative",
        "TextToSpeechStyle": "None"
      },
      "Transitions": {
        "NextAction": "GREETING_UUID",
        "Errors": [
          { "NextAction": "GREETING_UUID", "ErrorType": "NoMatchingError" }
        ]
      }
    },
    {
      "Identifier": "GREETING_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "Welcome to ..." },
      "Transitions": { "NextAction": "LEX_MENU_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "LEX_MENU_UUID",
      "Type": "ConnectParticipantWithLexBot",
      "Parameters": {
        "Text": "Please say or press 1 for providers, or say or press 2 for patients.",
        "LexV2Bot": {
          "AliasArn": "arn:aws:lex:REGION:ACCOUNT:bot-alias/BOTID/ALIASID"
        }
      },
      "Transitions": {
        "NextAction": "ERROR_UUID",
        "Conditions": [
          { "NextAction": "PROVIDER_UUID", "Condition": { "Operator": "Equals", "Operands": ["ProviderIntent"] } },
          { "NextAction": "PATIENT_UUID", "Condition": { "Operator": "Equals", "Operands": ["PatientIntent"] } }
        ],
        "Errors": [
          { "NextAction": "ERROR_UUID", "ErrorType": "NoMatchingCondition" },
          { "NextAction": "ERROR_UUID", "ErrorType": "NoMatchingError" }
        ]
      }
    }
  ]
}
```

## Template: Nova Sonic Speech-to-Speech IVR

Use when the Lex bot is configured with Speech-to-Speech: Amazon Nova Sonic and Q Connect AI agent. The `Generative` engine enables Nova Sonic's expressive voice output. **The `CreateWisdomSession` block is required** to create a Q Connect session before the Lex bot can hand off to the AI agent.

```json
{
  "Version": "2019-10-30",
  "StartAction": "SET_VOICE_UUID",
  "Metadata": {
    "entryPointPosition": { "x": 39, "y": 40 },
    "ActionMetadata": {
      "SET_VOICE_UUID": { "position": { "x": 250, "y": 200 } },
      "GREETING_UUID": { "position": { "x": 550, "y": 200 } },
      "WISDOM_SESSION_UUID": { "position": { "x": 850, "y": 200 } },
      "LEX_S2S_UUID": { "position": { "x": 1150, "y": 200 } },
      "QIC_DONE_UUID": { "position": { "x": 1500, "y": 50 } },
      "FALLBACK_UUID": { "position": { "x": 1500, "y": 250 } },
      "ERROR_UUID": { "position": { "x": 1500, "y": 450 } },
      "DISCONNECT_UUID": { "position": { "x": 1850, "y": 200 } }
    }
  },
  "Actions": [
    {
      "Identifier": "SET_VOICE_UUID",
      "Type": "UpdateContactTextToSpeechVoice",
      "Parameters": {
        "TextToSpeechVoice": "Matthew",
        "TextToSpeechEngine": "Generative",
        "TextToSpeechStyle": "None"
      },
      "Transitions": {
        "NextAction": "GREETING_UUID",
        "Errors": [
          { "NextAction": "GREETING_UUID", "ErrorType": "NoMatchingError" }
        ]
      }
    },
    {
      "Identifier": "GREETING_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "Welcome! I'm your AI assistant powered by Nova Sonic." },
      "Transitions": { "NextAction": "WISDOM_SESSION_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "WISDOM_SESSION_UUID",
      "Type": "CreateWisdomSession",
      "Parameters": {
        "WisdomAssistantArn": "arn:aws:wisdom:REGION:ACCOUNT:assistant/ASSISTANT_ID"
      },
      "Transitions": {
        "NextAction": "LEX_S2S_UUID",
        "Errors": [
          { "NextAction": "ERROR_UUID", "ErrorType": "NoMatchingError" }
        ],
        "Conditions": []
      }
    },
    {
      "Identifier": "LEX_S2S_UUID",
      "Type": "ConnectParticipantWithLexBot",
      "Parameters": {
        "Text": "How can I help you today?",
        "LexV2Bot": {
          "AliasArn": "arn:aws:lex:REGION:ACCOUNT:bot-alias/BOTID/ALIASID"
        }
      },
      "Transitions": {
        "NextAction": "FALLBACK_UUID",
        "Conditions": [
          { "NextAction": "QIC_DONE_UUID", "Condition": { "Operator": "Equals", "Operands": ["QinConnectIntent"] } }
        ],
        "Errors": [
          { "NextAction": "FALLBACK_UUID", "ErrorType": "NoMatchingCondition" },
          { "NextAction": "ERROR_UUID", "ErrorType": "NoMatchingError" }
        ]
      }
    },
    {
      "Identifier": "QIC_DONE_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "Thanks for chatting! Goodbye." },
      "Transitions": { "NextAction": "DISCONNECT_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "FALLBACK_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "I'm sorry, I didn't understand that. Let me transfer you to an agent." },
      "Transitions": { "NextAction": "DISCONNECT_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "ERROR_UUID",
      "Type": "MessageParticipant",
      "Parameters": { "Text": "We're experiencing a technical issue. Please try again later." },
      "Transitions": { "NextAction": "DISCONNECT_UUID", "Errors": [], "Conditions": [] }
    },
    {
      "Identifier": "DISCONNECT_UUID",
      "Type": "DisconnectParticipant",
      "Parameters": {},
      "Transitions": {}
    }
  ]
}
```

**Key points for Nova Sonic IVRs:**
- `TextToSpeechEngine` MUST be `"Generative"` — this enables Nova Sonic
- Use a Nova Sonic-compatible voice: `Matthew`, `Amy`, `Olivia`, `Lupe`
- The `LexV2Bot.AliasArn` must point to a bot alias with Speech-to-Speech: Amazon Nova Sonic enabled at the locale level
- Nova Sonic handles interruptions (barge-in) natively — no special config needed
- Only available in `us-east-1` and `us-west-2`

## Common Pitfalls

1. **Forgetting `StoreInput` parameter** — Must be `"False"` (string) for menu selections
2. **Using integers instead of strings** — `InputTimeLimitSeconds` must be a string: `"5"` not `5`
3. **Missing error transitions** — Every `GetParticipantInput` needs `InputTimeLimitExceeded`, `NoMatchingCondition`, and `NoMatchingError`
4. **Wrong condition operands for Lex** — Use intent names (e.g., `"ProviderIntent"`), not DTMF digits
5. **Circular references** — A block pointing back to itself without a loop guard will cause infinite loops
