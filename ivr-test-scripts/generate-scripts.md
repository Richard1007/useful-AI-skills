# Generate Test Scripts

## Dialogue Format

Every scenario uses this concise format:

```markdown
## S1: [Descriptive Name]
**Test when:** [When to test — e.g., Mon-Fri, Sat-Sun]

System: "[exact system prompt text]"
Caller: [Action — e.g., Press 1 / Say "check my status" / Enter 123456789#]
System: "[next prompt text]"
Caller: [Next action]
System: "[response text]"
System: Transfers to [queue] / Call disconnects
```

## Rules

### System Lines
- Quote the exact TTS text from the flow JSON in double quotes
- For actions (not speech), describe without quotes: `System: Transfers to Tier 1 queue`
- For repeated prompt sequences, abbreviate after first full occurrence: `System: EN greeting + auth prompt`
- Always include the final outcome: transfer, disconnect, or callback confirmation

### Caller Lines
Keep caller actions short and direct:

**DTMF:**
- `Caller: Press 1`
- `Caller: Enter 123456789#`
- `Caller: Enter 01151990#`

**Voice:**
- `Caller: Say "check my authorization status"`
- `Caller: Say "I'd like to speak with a representative please"`
- `Caller: Say "um... I think I need to... check my authorization?"`

**Silence / timeout:**
- `Caller: Silent for 10+ seconds`

**Invalid:**
- `Caller: Say "blue elephant sandwich"`
- `Caller: Press 7`
- `Caller: Press # or *`

### Test When Lines
Each scenario must state when it can be tested:
- `**Test when:** Mon-Fri` — for business-hours paths
- `**Test when:** Sat-Sun` — for after-hours paths
- `**Test when:** Mon-Fri (after successful auth)` — for paths that require prior steps
- `**Test when:** Mon-Fri (requires Tier 1 queue at capacity)` — for conditional paths

### Scenario Naming
Use short IDs and descriptive names:
- `## S1: EN - Check Auth Status (DTMF)`
- `## S8: ES - Check Auth Status (DTMF)`
- `## S11: Auth Retry - First Failure Then Success`
- `## S18: After Hours - EN Callback`

## Scenario Categories

### 1. Happy Path — DTMF
One scenario per valid end-to-end path using keypad input.

### 2. Happy Path — Voice
Same paths but caller uses natural speech. Vary phrasings across scenarios.

### 3. Mixed Input
DTMF at one step, voice at another. Tests input method switching.

### 4. Error — Invalid Input
Invalid DTMF or unrecognized speech at each menu level.

### 5. Error — Timeout / Silence
Caller stays silent at each menu level.

### 6. Edge Cases
Special keys (#, *, 0), hesitant speech, out-of-domain phrases ("help"), retry limits.

### 7. After Hours
All after-hours paths: callback, voicemail, timeout, invalid input.

### 8. Queue Overflow
Tier escalation, both-tiers-full callback, end-call options.

## Abbreviation Rules

After the first scenario shows the full prompt text for a repeated sequence, later scenarios may abbreviate:

| First occurrence (full text) | Abbreviated form |
|-----|-----|
| `System: "For English, press 1. Para español, oprima el número 2."` | `System: Language selection` |
| `System: "Thank you for calling..." + auth prompt` | `System: EN greeting + auth prompt` |
| `System: "Gracias por llamar..." + auth prompt` | `System: ES greeting + auth prompt` |
| Full auth exchange (member ID + DOB) | `Caller: Enter 987654321# then 07041985#` |

## Voice Utterance Variety

For voice scenarios, vary the caller's phrasing:

| Type | Example |
|------|---------|
| Primary keyword | Say "check my authorization status" |
| Full phrase | Say "I'd like to speak with a representative please" |
| Casual | Say "call me back later" |
| Hesitant | Say "um... I think I need to... check my authorization?" |

## Output Structure

```markdown
# [Flow Name] - Test Scripts

**Flow ID:** [flow-id]
**Instance ID:** [instance-id]
**Bots:** [bot names]

---

## S1: [Name]
**Test when:** [condition]

System: "[prompt]"
Caller: [action]
System: "[response]"
...

---

## S2: [Name]
...
```

**Do NOT include** Coverage Matrix, Bot Utterance Reference, or any other appendix sections. The file contains ONLY the metadata header and the scenario scripts.

## Coverage Checklist (internal — do NOT output)

After generating all scenarios, internally verify before finalizing:
- Every valid end-to-end path has a DTMF scenario
- Every valid end-to-end path has a Voice scenario (if Lex is used)
- Invalid input tested at every menu level
- Timeout tested at every menu level
- At least one mixed-input scenario exists
- After-hours / queue overflow paths covered (if applicable)
