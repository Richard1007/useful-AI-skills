# Chat API + Participant API Reference

This document provides the complete API reference for testing Amazon Connect IVR flows via the Chat API. This is the same mechanism used by the Connect console's "Test Chat" feature.

**Source:** AWS Connect API Reference + AWS Connect Participant Service API Reference

---

## API Call Sequence

```
1. StartChatContact          → Creates chat contact, returns ParticipantToken + ContactId
2. CreateParticipantConnection → Exchanges ParticipantToken for ConnectionToken
3. GetTranscript              → Poll for system messages (greeting, prompts)
4. SendMessage               → Send customer text ("provider", "1", etc.)
5. GetTranscript              → Poll for new system messages after each send
6. ... repeat 4-5 for each interaction step ...
7. DisconnectParticipant     → Clean up (end chat session)
```

**Authentication model:**
- Step 1 uses **SigV4** (standard AWS credentials)
- Steps 2-7 use **ConnectionToken** (returned by Step 2) — NOT SigV4

---

## Operations

### 1. StartChatContact

Creates a new chat contact that routes through the specified contact flow.

**Service:** Amazon Connect (`connect`)
**Endpoint:** `POST /contact/chat`

```python
response = connect_client.start_chat_contact(
    InstanceId="7d261e94-17bc-4f3e-96f7-f9b7541ce479",
    ContactFlowId="846ec553-a005-41c0-8341-xxxxxxxxxxxx",
    ParticipantDetails={"DisplayName": "IVR-Test-Runner"},
    SupportedMessagingContentTypes=["text/plain"],
)

participant_token = response["ParticipantToken"]
contact_id = response["ContactId"]
```

**Key parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `InstanceId` | Yes | Connect instance ID |
| `ContactFlowId` | Yes | Contact flow to test |
| `ParticipantDetails.DisplayName` | Yes | Name shown in transcript |
| `SupportedMessagingContentTypes` | No | `["text/plain"]` for IVR testing |
| `Attributes` | No | Initial contact attributes (key-value pairs) |

**Returns:**

| Field | Description |
|-------|-------------|
| `ParticipantToken` | Token to create participant connection (expires in 12 hours) |
| `ContactId` | Unique identifier for this chat contact |

**AWS CLI equivalent:**
```bash
aws connect start-chat-contact \
  --instance-id 7d261e94-17bc-4f3e-96f7-f9b7541ce479 \
  --contact-flow-id 846ec553-a005-41c0-8341-xxxxxxxxxxxx \
  --participant-details '{"DisplayName":"IVR-Test-Runner"}' \
  --profile haohai --region us-west-2
```

**Docs:** [StartChatContact API](https://docs.aws.amazon.com/connect/latest/APIReference/API_StartChatContact.html)

---

### 2. CreateParticipantConnection

Exchanges the `ParticipantToken` for a `ConnectionToken` used in all subsequent calls.

**Service:** Amazon Connect Participant Service (`connectparticipant`)
**Endpoint:** `POST /participant/connection`

```python
response = participant_client.create_participant_connection(
    ParticipantToken=participant_token,
    Type=["CONNECTION_CREDENTIALS"],
    ConnectParticipant=True,
)

connection_token = response["ConnectionCredentials"]["ConnectionToken"]
```

**Key parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `ParticipantToken` | Yes | From `StartChatContact` response |
| `Type` | Yes | `["CONNECTION_CREDENTIALS"]` for polling-based approach |
| `ConnectParticipant` | No | `True` to auto-connect participant |

**Returns:**

| Field | Description |
|-------|-------------|
| `ConnectionCredentials.ConnectionToken` | Token for all subsequent Participant API calls (expires 24 hours) |
| `ConnectionCredentials.Expiry` | Token expiration timestamp |

**Important:** The `ConnectParticipant=True` flag ensures the participant is connected immediately. Without it, you'd need a separate `ConnectParticipant` call.

**Docs:** [CreateParticipantConnection API](https://docs.aws.amazon.com/connect-participant/latest/APIReference/API_CreateParticipantConnection.html)

---

### 3. SendMessage

Sends a text message from the customer to the contact flow. This is how the customer "speaks" or "presses digits."

**Service:** Amazon Connect Participant Service (`connectparticipant`)
**Endpoint:** `POST /participant/message`

```python
participant_client.send_message(
    ConnectionToken=connection_token,
    ContentType="text/plain",
    Content="provider",
)
```

**Key parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `ConnectionToken` | Yes | From `CreateParticipantConnection` response |
| `ContentType` | Yes | `"text/plain"` for IVR testing |
| `Content` | Yes | The text to send (e.g., `"provider"`, `"1"`, `"referrals"`) |

**How text maps to flow behavior:**
- `"1"` → Lex bot matches intent with `"1"` as sample utterance (simulates DTMF)
- `"provider"` → Lex bot matches intent with `"provider"` as sample utterance (simulates voice)
- `"I'd like to order pizza"` → Lex bot matches `FallbackIntent` (tests error path)

**Returns:** `MessageId` and `AbsoluteTime` — generally not needed for testing.

**Docs:** [SendMessage API](https://docs.aws.amazon.com/connect-participant/latest/APIReference/API_SendMessage.html)

---

### 4. GetTranscript

Retrieves the chat transcript. Used to capture system messages (prompts, confirmations, error messages).

**Service:** Amazon Connect Participant Service (`connectparticipant`)
**Endpoint:** `POST /participant/transcript`

```python
response = participant_client.get_transcript(
    ConnectionToken=connection_token,
    SortOrder="ASCENDING",
    MaxResults=100,
)

for item in response["Transcript"]:
    role = item.get("ParticipantRole")      # "SYSTEM", "CUSTOMER", or "AGENT"
    content = item.get("Content", "")
    msg_type = item.get("Type")             # "MESSAGE", "EVENT", etc.
    timestamp = item.get("AbsoluteTime")
```

**Key parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `ConnectionToken` | Yes | From `CreateParticipantConnection` |
| `SortOrder` | No | `"ASCENDING"` (oldest first) or `"DESCENDING"` |
| `MaxResults` | No | 1-100 (default varies) |
| `NextToken` | No | For pagination |

**Transcript item fields:**

| Field | Description |
|-------|-------------|
| `ParticipantRole` | `"SYSTEM"` (flow prompts), `"CUSTOMER"` (your messages), `"AGENT"` (if transferred) |
| `Content` | Message text |
| `Type` | `"MESSAGE"` (text), `"EVENT"` (join/leave), `"ATTACHMENT"` |
| `AbsoluteTime` | ISO 8601 timestamp |
| `Id` | Unique message ID |

**Filtering for test assertions:**
Only `SYSTEM` messages with `Type="MESSAGE"` and non-empty `Content` are relevant for test assertions. Customer messages and events should be ignored.

**Pagination:** If `NextToken` is present in the response, there are more items. Loop until `NextToken` is absent.

**Docs:** [GetTranscript API](https://docs.aws.amazon.com/connect-participant/latest/APIReference/API_GetTranscript.html)

---

### 5. DisconnectParticipant

Ends the chat session. Always call this for cleanup, even if the test fails.

**Service:** Amazon Connect Participant Service (`connectparticipant`)
**Endpoint:** `POST /participant/disconnect`

```python
participant_client.disconnect_participant(
    ConnectionToken=connection_token,
)
```

**Key parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `ConnectionToken` | Yes | From `CreateParticipantConnection` |

**Important:** Always wrap in try/except — the session may have already ended (flow disconnected the contact).

**Docs:** [DisconnectParticipant API](https://docs.aws.amazon.com/connect-participant/latest/APIReference/API_DisconnectParticipant.html)

---

## IAM Permissions

### Minimum Required Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StartChat",
      "Effect": "Allow",
      "Action": [
        "connect:StartChatContact"
      ],
      "Resource": "arn:aws:connect:*:*:instance/*/contact-flow/*"
    },
    {
      "Sid": "ParticipantService",
      "Effect": "Allow",
      "Action": [
        "connect-participant:CreateParticipantConnection",
        "connect-participant:SendMessage",
        "connect-participant:GetTranscript",
        "connect-participant:DisconnectParticipant"
      ],
      "Resource": "*"
    }
  ]
}
```

**Why `Resource: "*"` for Participant Service?** The Participant Service APIs authenticate using `ConnectionToken`, not IAM resource ARNs. The IAM policy just authorizes the account to call the APIs — access scoping is done by the token itself.

### Restrict to Specific Instance

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StartChat",
      "Effect": "Allow",
      "Action": ["connect:StartChatContact"],
      "Resource": "arn:aws:connect:us-west-2:123456789012:instance/7d261e94-17bc-4f3e-96f7-f9b7541ce479/contact-flow/*"
    },
    {
      "Sid": "ParticipantService",
      "Effect": "Allow",
      "Action": [
        "connect-participant:CreateParticipantConnection",
        "connect-participant:SendMessage",
        "connect-participant:GetTranscript",
        "connect-participant:DisconnectParticipant"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Flow Block Chat Compatibility

This table defines which flow blocks work in chat mode. Use it to identify untestable paths.

| Flow Block | Chat Support | Behavior in Chat | Source |
|-----------|-------------|-----------------|--------|
| `ConnectParticipantWithLexBot` | ✅ Works | Customer types text → Lex matches intent | [Get customer input docs](https://docs.aws.amazon.com/connect/latest/adminguide/get-customer-input.html) |
| `GetParticipantInput` (DTMF-only, no Lex) | ❌ Error branch | Takes Error branch — no text input accepted | [Get customer input docs](https://docs.aws.amazon.com/connect/latest/adminguide/get-customer-input.html) |
| `StoreUserInput` | ❌ Error branch | Takes Error branch — voice-only block | [Store customer input docs](https://docs.aws.amazon.com/connect/latest/adminguide/store-customer-input.html) |
| `MessageParticipant` (TTS text) | ✅ Works | Text sent as chat message | [Play prompt docs](https://docs.aws.amazon.com/connect/latest/adminguide/play.html) |
| `MessageParticipant` (audio-only) | ❌ Silent | No message in transcript — audio not supported in chat | [Play prompt docs](https://docs.aws.amazon.com/connect/latest/adminguide/play.html) |
| `InvokeAWSLambdaFunction` | ✅ Works | Lambda executes normally | |
| `UpdateContactAttributes` | ✅ Works | Attributes set/read correctly | |
| `TransferContactToQueue` | ✅ Works | Transfer executes | |
| `CheckContactAttributes` | ✅ Works | Condition evaluation works | |
| `Compare` | ✅ Works | Comparison logic works | |
| `Distribute` | ✅ Works | Percentage routing works | |
| `DisconnectParticipant` | ✅ Works | Chat session ends | |

---

## Throttling and Rate Limits

### Observed Limits

| API | Approximate Limit | Notes |
|-----|-------------------|-------|
| `StartChatContact` | ~10/second | Shared with production chat traffic |
| `CreateParticipantConnection` | ~50/second | Per-connection, rarely hit |
| `SendMessage` | ~10/second per connection | Per chat session |
| `GetTranscript` | ~100/second | Read-heavy, generous limits |
| `DisconnectParticipant` | ~50/second | Cleanup, rarely hit |

### Mitigation

- **Between scenarios:** 2-second cooldown (configurable via `SCENARIO_COOLDOWN`)
- **Between messages in a scenario:** 2-second turn wait (configurable via `TURN_WAIT`)
- **On throttle (HTTP 429):** Exponential backoff — boto3 handles this automatically with default retry config

### Important: Production Impact

`StartChatContact` creates **real contacts** that count toward your Connect instance's concurrent chat limit. For large test suites (50+ scenarios), consider:

1. Running during off-peak hours
2. Monitoring the Connect dashboard for concurrent contact count
3. Ensuring `DisconnectParticipant` is always called (cleanup)

---

## AWS CLI Debug Commands

Useful for troubleshooting when the script fails:

### Verify Flow Exists

```bash
aws connect describe-contact-flow \
  --instance-id 7d261e94-17bc-4f3e-96f7-f9b7541ce479 \
  --contact-flow-id 846ec553-a005-41c0-8341-xxxxxxxxxxxx \
  --profile haohai --region us-west-2 \
  --query 'ContactFlow.{Name:Name,State:State,Type:Type}'
```

### List Contact Flows

```bash
aws connect list-contact-flows \
  --instance-id 7d261e94-17bc-4f3e-96f7-f9b7541ce479 \
  --contact-flow-types CONTACT_FLOW \
  --profile haohai --region us-west-2 \
  --query 'ContactFlowSummaryList[*].{Name:Name,Id:Id}'
```

### Manual Chat Test (CLI)

Step-by-step to test a single interaction via CLI:

```bash
# 1. Start chat
RESPONSE=$(aws connect start-chat-contact \
  --instance-id 7d261e94-17bc-4f3e-96f7-f9b7541ce479 \
  --contact-flow-id 846ec553-a005-41c0-8341-xxxxxxxxxxxx \
  --participant-details '{"DisplayName":"CLI-Test"}' \
  --profile haohai --region us-west-2 \
  --output json)

PARTICIPANT_TOKEN=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['ParticipantToken'])")

# 2. Create connection
CONN_RESPONSE=$(aws connectparticipant create-participant-connection \
  --participant-token "$PARTICIPANT_TOKEN" \
  --type CONNECTION_CREDENTIALS \
  --region us-west-2 \
  --output json)

CONNECTION_TOKEN=$(echo $CONN_RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['ConnectionCredentials']['ConnectionToken'])")

# 3. Wait for greeting
sleep 3

# 4. Get transcript (see system messages)
aws connectparticipant get-transcript \
  --connection-token "$CONNECTION_TOKEN" \
  --sort-order ASCENDING \
  --region us-west-2 \
  --query 'Transcript[?ParticipantRole==`SYSTEM`].Content'

# 5. Send a message
aws connectparticipant send-message \
  --connection-token "$CONNECTION_TOKEN" \
  --content-type "text/plain" \
  --content "provider" \
  --region us-west-2

# 6. Wait + get transcript again
sleep 3
aws connectparticipant get-transcript \
  --connection-token "$CONNECTION_TOKEN" \
  --sort-order ASCENDING \
  --region us-west-2 \
  --query 'Transcript[?ParticipantRole==`SYSTEM`].Content'

# 7. Disconnect
aws connectparticipant disconnect-participant \
  --connection-token "$CONNECTION_TOKEN" \
  --region us-west-2
```

### Check CloudWatch Logs

```bash
# Flow execution logs
aws logs tail "/aws/connect/7d261e94-17bc-4f3e-96f7-f9b7541ce479" \
  --follow --profile haohai --region us-west-2

# Filter for specific contact
aws logs filter-log-events \
  --log-group-name "/aws/connect/7d261e94-17bc-4f3e-96f7-f9b7541ce479" \
  --filter-pattern "CONTACT_ID" \
  --profile haohai --region us-west-2
```

---

## Troubleshooting Test Behavior

### No System Messages in Transcript

**Symptom:** `GetTranscript` returns empty or only customer messages.

**Causes:** (1) Flow uses audio-only prompts -- chat can't render audio. (2) Flow branches by channel and skips chat -- check `CheckContactAttributes` on `$.Channel`. (3) Polling too early -- flow hasn't finished processing.

**Fix:** Verify Play prompt blocks use "Text-to-speech or chat text". Check for channel-specific branching. Increase `INITIAL_WAIT` or `POLL_TIMEOUT`.

### Scenario Takes Error Path

**Symptom:** System messages match the error/fallback path instead of expected path.

**Causes:** (1) Flow uses `GetParticipantInput` (DTMF-only) -- takes Error branch in chat. (2) Lex bot missing the typed text as sample utterance. (3) Flow expects contact attributes not present in chat.

**Fix:** Check if block is `GetParticipantInput` vs `ConnectParticipantWithLexBot`. Verify Lex utterances. Pass attributes via `StartChatContact.Attributes`.

### Chat Session Disconnects Immediately

**Symptom:** `CreateParticipantConnection` succeeds but transcript is empty.

**Causes:** (1) Flow immediately disconnects on chat channel. (2) Wrong or unpublished flow ID. (3) Flow error routes to disconnect.

**Fix:** Test manually in Connect console's Test Chat. Verify flow ID with `aws connect describe-contact-flow`. Check CloudWatch Logs.

### Timeout Waiting for Messages

**Symptom:** Script times out waiting for system messages after sending text.

**Causes:** (1) Lex bot chat timeout not configured. (2) Lambda taking too long. (3) Flow entered infinite loop.

**Fix:** Increase `POLL_TIMEOUT`. Check Lambda in CloudWatch Logs. Review flow for loops or missing error handling.

---

## Cross-References

- **SKILL.md** — Skill entry point, workflow, results format
- **execution-engine.md** — Script template, scenario data format, generation rules, output checklist
- **ivr-test-scripts skill** — Generates test scenario documents consumed by this skill
- **amazon-connect-ivr skill** — Flow design rules that affect chat testability
