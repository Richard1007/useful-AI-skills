# Amazon Connect IVR AI Organization

This is a self-improving AI organization for building, testing, and refining Amazon Connect IVR flows with Lex bots.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          MANAGER AGENT (Supervisor)                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ • Orchestrates the 3 skill agents                                       │  │
│  │ • Runs improvement loops until tests pass                                │  │
│  │ • Tracks component test coverage                                        │  │
│  │ • Decides when to escalate to human review                               │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
         ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
         │   IVR SKILL  │  │  LEX BOT     │  │   TEST       │
         │   (Skill 1)  │  │  (Skill 2)  │  │  (Skill 3)  │
         └──────────────┘  └──────────────┘  └──────────────┘
                │                 │                  │
                ▼                 ▼                  ▼
    Creates Contact    Creates Lex Bots     Generates Test
    Flow JSON          with Intents         Scripts
    (Connect IVR)      & Slots              (Twilio/VoIP)
```

## Self-Improving Loop

```
                    ┌─────────────────────────────────────┐
                    │         IMPROVEMENT LOOP            │
                    │                                     │
                    │  1. Create Component Test          │
                    │     (single IVR block)             │
                    │           │                        │
                    │           ▼                        │
                    │  2. Generate IVR + Lex + Test     │
                    │           │                        │
                    │           ▼                        │
                    │  3. Deploy to Connect              │
                    │           │                        │
                    │           ▼                        │
                    │  4. Self-Dial & Test              │
                    │     (Twilio API)                  │
                    │           │                        │
                    │           ▼                        │
                    │  5. Compare vs Expected           │
                    │           │                        │
                    │           ▼                        │
                    │     ┌───────┐                      │
                    │     │Pass? │                      │
                    │     └──┬───┘                      │
                    │   Yes/│\No                         │
                    │    Y  │ N                          │
                    │   ┌───┘└────────────┐             │
                    │   │                 │             │
                    │   ▼                 ▼             │
                    │  PASS             FIX:            │
                    │              Update Skill         │
                    │              Prompt/Pattern        │
                    │                  │                │
                    │                  └──────┐         │
                    │                         │         │
                    └─────────────────────────┘         │
                                                        
                    Loop until all components pass
```

## Component Test Matrix

| Component | Test Type | Expected Behavior |
|-----------|-----------|-------------------|
| PlayPrompt | Audio check | Plays correct prompt |
| GetInput | DTMF detection | Captures keypress |
| GetCustomerInput | Speech | Recognizes speech |
| Lex | Intent match | Routes to correct intent |
| Decision | Branch | Routes based on condition |
| Transfer | Call routing | Transfers correctly |
| Queue | Wait handling | Enqueues properly |
| Loop | Iteration | Loops correct times |
| Disconnect | End call | Terminates cleanly |

## Usage

### Basic Flow - Test Single Component

```typescript
// Manager creates a simple PlayPrompt test
task(
  category="ivr-builder",
  load_skills=["connect-ivr"],
  prompt="Create a test IVR for PlayPrompt component only. 
          Requirements:
          - Play prompt: 'Welcome to test'
          - Simple flow, one block only"
)

// Generate test script
task(
  category="test-generator",
  load_skills=["ivr-test-scripts"],
  prompt="Generate test script that verifies prompt plays correctly"
)

// Run test, analyze, improve
```

### Full Test Suite

```typescript
// Manager orchestrates full component coverage
task(
  category="manager",
  load_skills=["connect-ivr", "lex-bot", "ivr-test-scripts"],
  prompt="Test all Amazon Connect flow components.
          For each component:
          1. Create minimal IVR testing that component
          2. Generate test script
          3. Run test via Twilio
          4. If fail: analyze failure, improve skill
          5. Repeat until pass
          
          Track: { component: 'PlayPrompt', status: 'pass', attempts: 3 }"
)
```

## Skills Configuration

### connect-ivr Skill

```yaml
---
name: connect-ivr
description: Creates Amazon Connect Contact Flow JSON for IVR testing
mcp:
  aws-connect:
    command: npx
    args: ["-y", "@aws-sdk/client-connect"]
---

# Amazon Connect IVR Expert

You create minimal, isolated IVR flows for testing individual components.

## Component Patterns

### PlayPrompt Only
- Use: Play prompt, then disconnect
- Blocks: PlayPrompt → Disconnect

### GetInput (DTMF)
- Use: Get keypress, then play back
- Blocks: PlayPrompt → GetInput → PlayPrompt → Disconnect

### Lex Integration
- Use: Collect input, pass to Lex
- Blocks: PlayPrompt → GetCustomerInput → Lex → Disconnect

### Queue Flow
- Use: Queue caller, play music
- Blocks: PlayPrompt → SetQueue → PlayMusic → Disconnect
```

### lex-bot Skill

```yaml
---
name: lex-bot
description: Creates Amazon Lex V2 bots for Connect IVR integration
mcp:
  aws-lex:
    command: npx
    args: ["-y", "@aws-sdk/client-lexv2-models"]
---

# Amazon Lex Bot Expert

You create minimal Lex bots for IVR testing.

## Bot Patterns

### Simple Intent Bot
- Single intent
- One slot
- Confirmation prompt optional

### Menu Bot
- Multiple intents (one per menu option)
- No slots needed
- Utterances: "Sales", "Support", "Billing"

### Slot Collection Bot
- Single intent with required slots
- Multiple slot types
- Validation via Lambda
```

### ivr-test-scripts Skill

```yaml
---
name: ivr-test-scripts
description: Generates executable test scripts for IVR verification
mcp:
  twilio:
    command: npx
    args: ["-y", "twilio"]
---

# IVR Test Script Expert

You generate automated test scripts that:
1. Initiate call to IVR phone number
2. Navigate menus via DTMF/speech
3. Verify expected prompts play
4. Check routing logic
5. Detect failures

## Test Framework

Use Twilio or similar VoIP API:
- Call Connect phone number
- Wait for prompt
- Send DTMF or speak
- Capture response
- Compare vs expected
- Log pass/fail with details
```

## File Structure

```
project/
├── skills/
│   ├── connect-ivr/
│   │   ├── SKILL.md
│   │   └── examples/
│   │       ├── play-prompt.json
│   │       ├── get-input.json
│   │       └── queue-flow.json
│   ├── lex-bot/
│   │   ├── SKILL.md
│   │   └── examples/
│   │       ├── simple-menu.json
│   │       └── slot-collection.json
│   ├── ivr-test-scripts/
│   │   ├── SKILL.md
│   │   └── templates/
│   │       ├── twilio-test.js
│   │       └── verify-prompts.js
│   └── manager/
│       └── SKILL.md
├── tests/
│   └── component-results.json
└── flows/
    └── deployed/
```
