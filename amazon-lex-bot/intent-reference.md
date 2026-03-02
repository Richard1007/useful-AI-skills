# Intent and Utterance Reference

## Standard Intent Patterns for IVR Menus

### Caller Type Identification

#### ProviderIntent
```json
[
  {"utterance": "provider"},
  {"utterance": "I'm a provider"},
  {"utterance": "I am a provider"},
  {"utterance": "healthcare provider"},
  {"utterance": "doctor"},
  {"utterance": "I'm a doctor"},
  {"utterance": "physician"},
  {"utterance": "I'm calling as a provider"},
  {"utterance": "provider calling"},
  {"utterance": "this is a provider"},
  {"utterance": "calling as a healthcare provider"},
  {"utterance": "I'm a healthcare professional"},
  {"utterance": "1"}
]
```

#### PatientIntent
```json
[
  {"utterance": "patient"},
  {"utterance": "I'm a patient"},
  {"utterance": "I am a patient"},
  {"utterance": "member"},
  {"utterance": "I'm a member"},
  {"utterance": "I'm calling as a patient"},
  {"utterance": "patient calling"},
  {"utterance": "this is a patient"},
  {"utterance": "I'm calling about my health"},
  {"utterance": "I need to speak to someone"},
  {"utterance": "I'm a client"},
  {"utterance": "2"}
]
```

### Healthcare Service Intents

#### AppointmentIntent
```json
[
  {"utterance": "appointment"},
  {"utterance": "schedule an appointment"},
  {"utterance": "book an appointment"},
  {"utterance": "I need an appointment"},
  {"utterance": "I want to schedule a visit"},
  {"utterance": "make an appointment"},
  {"utterance": "I need to see a doctor"},
  {"utterance": "schedule a visit"},
  {"utterance": "book a visit"},
  {"utterance": "set up an appointment"},
  {"utterance": "1"}
]
```

#### PrescriptionIntent
```json
[
  {"utterance": "prescription"},
  {"utterance": "refill my prescription"},
  {"utterance": "I need a refill"},
  {"utterance": "prescription refill"},
  {"utterance": "medication refill"},
  {"utterance": "I need my medication"},
  {"utterance": "refill"},
  {"utterance": "renew my prescription"},
  {"utterance": "I need to refill my medicine"},
  {"utterance": "medicine refill"},
  {"utterance": "2"}
]
```

#### BillingIntent
```json
[
  {"utterance": "billing"},
  {"utterance": "I have a billing question"},
  {"utterance": "billing inquiry"},
  {"utterance": "my bill"},
  {"utterance": "I need to pay my bill"},
  {"utterance": "payment"},
  {"utterance": "I have a question about my bill"},
  {"utterance": "insurance billing"},
  {"utterance": "account balance"},
  {"utterance": "pay my balance"},
  {"utterance": "3"}
]
```

#### TestResultsIntent
```json
[
  {"utterance": "test results"},
  {"utterance": "I want my test results"},
  {"utterance": "lab results"},
  {"utterance": "get my results"},
  {"utterance": "check my results"},
  {"utterance": "I need my lab results"},
  {"utterance": "blood test results"},
  {"utterance": "results"},
  {"utterance": "my test results"},
  {"utterance": "check on my labs"},
  {"utterance": "1"}
]
```

#### ReferralIntent
```json
[
  {"utterance": "referral"},
  {"utterance": "I need a referral"},
  {"utterance": "get a referral"},
  {"utterance": "specialist referral"},
  {"utterance": "refer me to a specialist"},
  {"utterance": "I need to see a specialist"},
  {"utterance": "referral request"},
  {"utterance": "submit a referral"},
  {"utterance": "request a referral"},
  {"utterance": "2"}
]
```

#### PriorAuthIntent
```json
[
  {"utterance": "prior authorization"},
  {"utterance": "prior auth"},
  {"utterance": "I need a prior authorization"},
  {"utterance": "authorization"},
  {"utterance": "pre authorization"},
  {"utterance": "submit a prior auth"},
  {"utterance": "check prior authorization status"},
  {"utterance": "auth request"},
  {"utterance": "3"}
]
```

### General IVR Intents

#### SpeakToAgentIntent
```json
[
  {"utterance": "agent"},
  {"utterance": "speak to an agent"},
  {"utterance": "talk to someone"},
  {"utterance": "representative"},
  {"utterance": "I want to talk to a person"},
  {"utterance": "human"},
  {"utterance": "operator"},
  {"utterance": "live agent"},
  {"utterance": "connect me to an agent"},
  {"utterance": "transfer me"},
  {"utterance": "0"}
]
```

#### RepeatIntent
```json
[
  {"utterance": "repeat"},
  {"utterance": "say that again"},
  {"utterance": "repeat the options"},
  {"utterance": "what were the options"},
  {"utterance": "I didn't hear that"},
  {"utterance": "can you repeat"},
  {"utterance": "please repeat"},
  {"utterance": "again"},
  {"utterance": "*"}
]
```

## Utterance Design Rules

### 1. Coverage Formula
For each intent, provide utterances covering:
- **Single keyword**: `"appointment"`
- **Noun phrase**: `"an appointment"`
- **Action phrase**: `"schedule an appointment"`
- **First person**: `"I need an appointment"`
- **Polite form**: `"I'd like to schedule an appointment"`
- **Casual form**: `"need to book something"`
- **Synonym variants**: `"visit"`, `"booking"`
- **DTMF digit**: `"1"` (if applicable)

### 2. DTMF Digit Assignment
When using DTMF alongside voice:
- Assign ONE digit per intent
- Include the digit as a sample utterance
- Keep digit assignment consistent across menu levels:
  - `0` = Speak to agent / Operator
  - `1-9` = Menu options
  - `*` = Repeat
  - `#` = Go back / Main menu

### 3. Avoiding Conflicts
- Never use the same utterance in two intents
- Avoid utterances that are subsets: if `"billing"` is in BillingIntent, don't put `"billing question"` in another intent
- Keep DTMF digits unique per bot (one digit = one intent)
