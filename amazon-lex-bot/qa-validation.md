# Lex Bot QA and Validation

**Assume there are problems. Your job is to find them.**

## Phase 1: Configuration Validation

### 1. Bot Exists and Is Available
```bash
aws lexv2-models describe-bot --bot-id $BOT_ID --profile PROFILE
# botStatus must be "Available"
```

### 2. Locale Is Built
```bash
aws lexv2-models describe-bot-locale \
  --bot-id $BOT_ID --bot-version DRAFT --locale-id en_US --profile PROFILE \
  --query 'botLocaleStatus' --output text
# Must be "Built"
```

### 3. All Intents Exist
```bash
aws lexv2-models list-intents \
  --bot-id $BOT_ID --bot-version DRAFT --locale-id en_US --profile PROFILE
# Verify all expected intents are listed
```

### 4. Utterances Are Populated
For each intent:
```bash
aws lexv2-models describe-intent \
  --bot-id $BOT_ID --bot-version DRAFT --locale-id en_US \
  --intent-id INTENT_ID --profile PROFILE \
  --query 'sampleUtterances'
# Must have 8+ utterances per intent
```

### 5. No Overlapping Utterances
- [ ] No identical utterances across different intents
- [ ] No utterance that is a substring of another intent's utterance
- [ ] DTMF digits (1, 2, 3...) are each in only ONE intent

## Phase 2: Deployment Validation

### 1. Version Exists
```bash
aws lexv2-models list-bot-versions --bot-id $BOT_ID --profile PROFILE
# At least one version besides DRAFT
```

### 2. Alias Exists and Points to Correct Version
```bash
aws lexv2-models list-bot-aliases --bot-id $BOT_ID --profile PROFILE
# Alias must exist and reference the latest version
```

### 3. Associated with Connect
```bash
aws connect list-bots \
  --instance-id INSTANCE_ID --lex-version V2 --profile PROFILE
# Bot alias ARN must appear in the list
```

## Phase 3: Functional Testing

### Test via AWS CLI (Lex Runtime)
```bash
# Test voice-like input
aws lexv2-runtime recognize-text \
  --bot-id $BOT_ID \
  --bot-alias-id $ALIAS_ID \
  --locale-id en_US \
  --session-id "test-session-001" \
  --text "I'm a provider" \
  --profile PROFILE

# Expected: interpretations[0].intent.name = "ProviderIntent"
```

### Test each utterance pattern:
1. Single word: `"provider"`
2. Full phrase: `"I'm a provider"`
3. Synonym: `"doctor"`
4. DTMF digit: `"1"`
5. Ambiguous input: `"help"` (should match FallbackIntent if no HelpIntent)

### Verify each intent matches correctly:
```bash
for utterance in "provider" "I'm a provider" "doctor" "1" "patient" "I'm a patient" "member" "2"; do
  RESULT=$(aws lexv2-runtime recognize-text \
    --bot-id $BOT_ID \
    --bot-alias-id $ALIAS_ID \
    --locale-id en_US \
    --session-id "test-$(date +%s)" \
    --text "$utterance" \
    --profile PROFILE \
    --query 'interpretations[0].intent.name' --output text 2>/dev/null)
  echo "\"$utterance\" → $RESULT"
done
```

## Verification Loop

1. Create/update the bot intents and utterances
2. Build the bot locale
3. Wait for build to complete
4. Run Phase 1 configuration checks
5. Create version and alias (if not exists)
6. Run Phase 2 deployment checks
7. Run Phase 3 functional tests
8. **List all issues found**
9. Fix issues (update intents, add utterances, etc.)
10. **Rebuild** — any intent change requires a rebuild
11. Repeat until all tests pass

**Do not declare success until:**
- All Phase 1-3 checks pass
- Every expected utterance maps to the correct intent
- DTMF digits map correctly
- FallbackIntent catches unrecognized input
- Bot is associated with the Connect instance

## Common Fixes

| Issue | Fix |
|-------|-----|
| Wrong intent matched | Add more distinct utterances to the correct intent; remove ambiguous ones |
| FallbackIntent for valid input | Add more utterance variations; lower confidence threshold |
| Build fails | Check for duplicate utterances across intents |
| DTMF digit matches wrong intent | Ensure digit is only in one intent's utterances |
| Bot not in Connect | Run `associate-bot` command |
