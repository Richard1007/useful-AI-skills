# IVR AI Organization — Architecture Diagrams

---

## 1. Organization Structure

```mermaid
graph TD
    subgraph SUPERVISOR["MANAGER AGENT — Claude Opus 4.6"]
        direction TB
        MGR["Orchestrator<br/>Plans, delegates, evaluates, improves"]
    end

    subgraph SKILL_LAYER["SPECIALIST SKILLS — Claude Sonnet 4.5"]
        direction LR
        S1["amazon-connect-ivr<br/>IVR Builder"]
        S2["amazon-lex-bot<br/>Lex Bot Creator"]
        S3["ivr-test-scripts<br/>Test Script Generator"]
    end

    subgraph ARTIFACTS["PRODUCED ARTIFACTS"]
        direction LR
        A1["Contact Flow JSON<br/>Version 2019-10-30"]
        A2["Lex V2 Bot<br/>Intents, Slots, Utterances"]
        A3["Test Scripts<br/>100% path coverage"]
    end

    subgraph INFRA["AWS + TELEPHONY"]
        direction LR
        AWS["Amazon Connect<br/>Deploy IVR + Associate Bot"]
        TEL["Twilio / VoIP<br/>Programmatic Dialing"]
    end

    subgraph FEEDBACK["FEEDBACK LOOP"]
        direction TB
        RES{{"Test Result"}}
        PASS["PASS — Component Verified"]
        FAIL["FAIL — Root Cause Analysis"]
        FIX["Update Skill Knowledge"]
    end

    MGR -->|"delegates"| S1
    MGR -->|"delegates"| S2
    MGR -->|"delegates"| S3

    S1 --> A1
    S2 --> A2
    S3 --> A3

    A1 -->|"deploy"| AWS
    A2 -->|"associate"| AWS
    A3 -->|"execute"| TEL

    AWS <-->|"call"| TEL
    TEL --> RES

    RES -->|"all scenarios pass"| PASS
    RES -->|"any scenario fails"| FAIL
    FAIL --> FIX
    FIX -->|"re-generate"| MGR

    PASS -->|"next component"| MGR

    classDef supervisor fill:#1B2838,color:#E0E6ED,stroke:#4A90D9,stroke-width:2px
    classDef skill fill:#0D3B66,color:#E0E6ED,stroke:#4A90D9,stroke-width:1px
    classDef artifact fill:#1A3A2A,color:#A8D5BA,stroke:#2E8B57,stroke-width:1px
    classDef infra fill:#3B1A1A,color:#E8B4B4,stroke:#CD5C5C,stroke-width:1px
    classDef pass fill:#1A3A1A,color:#90EE90,stroke:#228B22,stroke-width:2px
    classDef fail fill:#3A1A1A,color:#FF6B6B,stroke:#DC143C,stroke-width:2px
    classDef fix fill:#3A2A1A,color:#FFD700,stroke:#DAA520,stroke-width:1px

    class MGR supervisor
    class S1,S2,S3 skill
    class A1,A2,A3 artifact
    class AWS,TEL infra
    class PASS pass
    class FAIL fail
    class FIX fix
```

---

## 2. Self-Improving Loop (No Max Attempts)

```mermaid
flowchart TD
    START(["Manager receives component to test"]) --> PLAN

    subgraph PLAN_PHASE["Phase 1: Plan"]
        PLAN["Identify component<br/>Design minimal test flow<br/>Determine if Lex needed"]
    end

    PLAN --> BUILD_IVR

    subgraph BUILD_PHASE["Phase 2: Build"]
        BUILD_IVR["Delegate to amazon-connect-ivr<br/>Create Contact Flow JSON"]
        BUILD_IVR --> LEX_CHECK{{"Lex<br/>needed?"}}
        LEX_CHECK -->|"Yes"| BUILD_LEX["Delegate to amazon-lex-bot<br/>Create bot, build, version, alias"]
        LEX_CHECK -->|"No"| BUILD_TEST
        BUILD_LEX --> BUILD_TEST["Delegate to ivr-test-scripts<br/>Generate test scenarios"]
    end

    BUILD_TEST --> DEPLOY

    subgraph DEPLOY_PHASE["Phase 3: Deploy"]
        DEPLOY["Deploy to AWS Connect<br/>aws connect create-contact-flow"]
        DEPLOY --> ASSOC{{"Lex bot?"}}
        ASSOC -->|"Yes"| ASSOCIATE["Associate bot with Connect<br/>aws connect associate-bot"]
        ASSOC -->|"No"| ASSIGN
        ASSOCIATE --> ASSIGN["Assign phone number to flow"]
    end

    ASSIGN --> DIAL

    subgraph TEST_PHASE["Phase 4: Test"]
        DIAL["Self-dial via Twilio"]
        DIAL --> EXEC["Execute each test scenario<br/>Send DTMF / Speak / Wait"]
        EXEC --> CAPTURE["Capture: prompts heard,<br/>routing results, timing"]
        CAPTURE --> COMPARE["Compare actual vs expected<br/>from test script"]
    end

    COMPARE --> RESULT{{"All scenarios<br/>pass?"}}

    RESULT -->|"YES"| TESTED["Mark component TESTED<br/>Record successful patterns<br/>Update coverage matrix"]
    TESTED --> NEXT{{"More components<br/>remaining?"}}
    NEXT -->|"Yes"| START
    NEXT -->|"No"| COMPLETE(["ALL COMPONENTS VERIFIED<br/>Skills fully refined"])

    RESULT -->|"NO"| ANALYZE

    subgraph IMPROVE_PHASE["Phase 5: Analyze and Improve"]
        ANALYZE["Identify failure type"]
        ANALYZE --> WHICH{{"Which skill<br/>caused failure?"}}
        WHICH -->|"IVR JSON wrong"| FIX_IVR["Update amazon-connect-ivr<br/>skill knowledge"]
        WHICH -->|"Lex bot wrong"| FIX_LEX["Update amazon-lex-bot<br/>skill knowledge"]
        WHICH -->|"Test script wrong"| FIX_TEST["Update ivr-test-scripts<br/>skill knowledge"]
        WHICH -->|"Infrastructure"| ESCALATE(["Escalate to human<br/>with full context"])
        FIX_IVR --> REBUILD["Re-generate with<br/>updated knowledge"]
        FIX_LEX --> REBUILD
        FIX_TEST --> REBUILD
    end

    REBUILD -->|"Loop back — no limit"| BUILD_IVR

    classDef phase fill:#1B2838,color:#E0E6ED,stroke:#4A90D9,stroke-width:1px
    classDef decision fill:#2C1A3A,color:#D8BFD8,stroke:#9370DB,stroke-width:2px
    classDef success fill:#1A3A1A,color:#90EE90,stroke:#228B22,stroke-width:2px
    classDef danger fill:#3A1A1A,color:#FF6B6B,stroke:#DC143C,stroke-width:2px
    classDef warn fill:#3A2A1A,color:#FFD700,stroke:#DAA520,stroke-width:1px
    classDef terminal fill:#0A0A0A,color:#FFFFFF,stroke:#FFFFFF,stroke-width:2px

    class PLAN_PHASE,BUILD_PHASE,DEPLOY_PHASE,TEST_PHASE,IMPROVE_PHASE phase
    class LEX_CHECK,ASSOC,RESULT,NEXT,WHICH decision
    class TESTED,COMPLETE success
    class ESCALATE danger
    class FIX_IVR,FIX_LEX,FIX_TEST warn
```

---

## 3. Component Test Coverage — All 60 Action Types

```mermaid
graph LR
    subgraph TIER1["TIER 1 — Core IVR : 25 verified"]
        direction TB

        subgraph INT["Interaction : 4"]
            I1["MessageParticipant"]
            I2["GetParticipantInput"]
            I3["ConnectParticipantWithLexBot"]
            I4["MessageParticipantIteratively"]
        end

        subgraph FC["Flow Control : 6"]
            F1["Compare"]
            F2["CheckHoursOfOperation"]
            F3["CheckMetricData"]
            F4["DistributeByPercentage"]
            F5["Wait"]
            F6["EndFlowExecution"]
        end

        subgraph RT["Routing : 5"]
            R1["UpdateContactTargetQueue"]
            R2["TransferContactToQueue"]
            R3["TransferToFlow"]
            R4["CreateCallbackContact"]
            R5["CreateTask"]
        end

        subgraph ST["Settings : 7"]
            S1["UpdateContactAttributes"]
            S2["UpdateContactTextToSpeechVoice"]
            S3["UpdateContactRecordingBehavior"]
            S4["UpdateContactEventHooks"]
            S5["UpdateContactRoutingBehavior"]
            S6["UpdateContactCallbackNumber"]
            S7["UpdatePreviousContactParticipantState"]
        end

        subgraph IG["Integration : 2"]
            G1["InvokeLambdaFunction"]
            G2["CreateWisdomSession"]
        end

        subgraph TM["Termination : 1"]
            T1["DisconnectParticipant"]
        end
    end

    subgraph TIER2["TIER 2 — Extended : 20 types"]
        direction TB
        E1["UpdateContactData"]
        E2["UpdateContactMediaStreamingBehavior"]
        E3["UpdateContactRecordingAndAnalyticsBehavior"]
        E4["TransferContactToAgent"]
        E5["DequeueContactAndTransferToQueue"]
        E6["TagContact / UnTagContact"]
        E7["PauseContact / ResumeContact"]
        E8["StoreInput"]
        E9["CheckStaffing / CheckVoiceId"]
        E10["InvokeFlowModule / EndFlowModuleExecution"]
        E11["UpdateFlowLoggingBehavior"]
        E12["ShowView"]
    end

    subgraph TIER3["TIER 3 — Advanced Integrations : 15 types"]
        direction TB
        A1["GetCustomerProfile"]
        A2["UpdateCustomerProfile"]
        A3["CreateCustomerProfile"]
        A4["AssociateContactToProfile"]
        A5["CreateCase / GetCase / UpdateCase"]
        A6["SetVoiceId / StartVoiceIdStream"]
        A7["TransferParticipantToThirdParty"]
    end

    classDef tier1 fill:#0D3B66,color:#E0E6ED,stroke:#4A90D9,stroke-width:1px
    classDef tier2 fill:#2C1A3A,color:#D8BFD8,stroke:#9370DB,stroke-width:1px
    classDef tier3 fill:#3B1A1A,color:#E8B4B4,stroke:#CD5C5C,stroke-width:1px
    classDef cat fill:#1B2838,color:#A0B4C8,stroke:#4A90D9,stroke-width:1px

    class I1,I2,I3,I4,F1,F2,F3,F4,F5,F6,R1,R2,R3,R4,R5,S1,S2,S3,S4,S5,S6,S7,G1,G2,T1 tier1
    class E1,E2,E3,E4,E5,E6,E7,E8,E9,E10,E11,E12 tier2
    class A1,A2,A3,A4,A5,A6,A7 tier3
    class INT,FC,RT,ST,IG,TM cat
```

---

## 4. Concrete Example — Testing GetParticipantInput

```mermaid
sequenceDiagram
    autonumber

    participant MGR as Manager Agent<br/>Claude Opus 4.6
    participant IVR as IVR Builder<br/>amazon-connect-ivr
    participant TST as Test Generator<br/>ivr-test-scripts
    participant AWS as Amazon Connect
    participant TEL as Twilio

    Note over MGR: Component: GetParticipantInput<br/>Goal: Verify DTMF menu routing

    rect rgb(13, 59, 102)
        Note right of MGR: BUILD PHASE
        MGR->>IVR: Create minimal flow:<br/>Prompt + GetParticipantInput (1=Sales, 2=Support)<br/>+ branch confirmations + disconnect
        IVR-->>MGR: Contact Flow JSON
        Note over MGR: Lex not needed for DTMF-only test
        MGR->>TST: Generate test script:<br/>S1: Press 1 -> hear sales<br/>S2: Press 2 -> hear support<br/>S3: Press 9 -> hear error<br/>S4: Silence -> hear timeout
        TST-->>MGR: Test Script (4 scenarios)
    end

    rect rgb(58, 26, 26)
        Note right of MGR: DEPLOY PHASE
        MGR->>AWS: aws connect create-contact-flow<br/>--content flow.json
        AWS-->>MGR: Flow ID + Phone Number
    end

    rect rgb(26, 58, 26)
        Note right of MGR: TEST PHASE
        loop Until all 4 scenarios pass
            MGR->>TEL: Dial +1-XXX-XXX-XXXX
            TEL->>AWS: Inbound call
            AWS-->>TEL: Press 1 for sales. Press 2 for support.
            TEL->>AWS: DTMF digit
            AWS-->>TEL: Branch confirmation prompt
            TEL-->>MGR: Actual result
            MGR->>MGR: Compare actual vs expected

            alt ALL PASS
                Note over MGR: Mark GetParticipantInput as TESTED
            else ANY FAIL
                MGR->>MGR: Analyze: which skill caused failure?
                alt IVR JSON issue
                    MGR->>IVR: Update skill: fix parameter/transition
                    IVR-->>MGR: Updated Contact Flow JSON
                else Test script issue
                    MGR->>TST: Update skill: fix timing/expectations
                    TST-->>MGR: Updated Test Script
                end
                MGR->>AWS: Re-deploy updated flow
                Note over MGR: Loop continues — no max attempts
            end
        end
    end

    MGR->>MGR: Update coverage matrix<br/>Move to next component
```

---

## 5. Test Progression Phases

```mermaid
graph TD
    subgraph P1["PHASE 1 — Individual Components"]
        direction LR
        P1A["Test each of 25 Tier 1<br/>action types in isolation"]
        P1B["Minimal flows:<br/>1-3 blocks each"]
        P1C["One component per flow"]
        P1A --- P1B --- P1C
    end

    subgraph P2["PHASE 2 — Component Combinations"]
        direction LR
        P2A["DTMF Menu + Queue Transfer"]
        P2B["Lex Bot + Lambda + Compare"]
        P2C["Hours Check + Callback"]
        P2D["Retry Loop Pattern"]
        P2A --- P2B --- P2C --- P2D
    end

    subgraph P3["PHASE 3 — Production IVR Flows"]
        direction LR
        P3A["Full customer service IVR<br/>Language + Auth + Routing"]
        P3B["Appointment scheduling<br/>Lex slot collection"]
        P3C["After-hours + Callback<br/>+ Voicemail"]
        P3A --- P3B --- P3C
    end

    P1 -->|"All 25 types pass"| P2
    P2 -->|"All combos pass"| P3
    P3 -->|"Production-ready"| DONE(["Skills fully validated<br/>Any AI using these skills<br/>can build correct IVRs"])

    classDef phase fill:#1B2838,color:#E0E6ED,stroke:#4A90D9,stroke-width:2px
    classDef done fill:#1A3A1A,color:#90EE90,stroke:#228B22,stroke-width:3px

    class P1,P2,P3 phase
    class DONE done
```
