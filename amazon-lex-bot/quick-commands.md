# Lex Bot CLI Quick Reference

## Create Bot

```bash
aws lexv2-models create-bot \
  --bot-name "MyBot" \
  --role-arn "arn:aws:iam::ACCOUNT:role/aws-service-role/lexv2.amazonaws.com/AWSServiceRoleForLexV2Bots_*" \
  --data-privacy '{"childDirected":false}' \
  --idle-session-ttl-in-seconds 300 \
  --profile PROFILE
```

## Create Locale

### Standard

```bash
aws lexv2-models create-bot-locale \
  --bot-id BOT_ID \
  --bot-version DRAFT \
  --locale-id en_US \
  --nlu-intent-confidence-threshold 0.40 \
  --profile PROFILE
```

### Nova Sonic (generative voice)

```bash
aws lexv2-models create-bot-locale \
  --bot-id BOT_ID \
  --bot-version DRAFT \
  --locale-id en_US \
  --nlu-intent-confidence-threshold 0.40 \
  --voice-settings '{"engine":"generative","voiceId":"Matthew"}' \
  --profile PROFILE
```

## Create Intent

```bash
aws lexv2-models create-intent \
  --bot-id BOT_ID \
  --bot-version DRAFT \
  --locale-id en_US \
  --intent-name "ProviderIntent" \
  --sample-utterances '[{"utterance":"provider"},{"utterance":"I am a provider"},{"utterance":"doctor"}]' \
  --profile PROFILE
```

## Build Bot Locale

```bash
aws lexv2-models build-bot-locale \
  --bot-id BOT_ID \
  --bot-version DRAFT \
  --locale-id en_US \
  --profile PROFILE
```

## Check Build Status

```bash
aws lexv2-models describe-bot-locale \
  --bot-id BOT_ID \
  --bot-version DRAFT \
  --locale-id en_US \
  --profile PROFILE \
  --query 'botLocaleStatus'
```

## Create Version

```bash
aws lexv2-models create-bot-version \
  --bot-id BOT_ID \
  --bot-version-locale-specification '{"en_US":{"sourceBotVersion":"DRAFT"}}' \
  --profile PROFILE
```

## Create Alias

Always include `--bot-alias-locale-settings` to enable the locale. Without it, the alias rejects requests.

```bash
aws lexv2-models create-bot-alias \
  --bot-id BOT_ID \
  --bot-alias-name "LiveAlias" \
  --bot-version "1" \
  --bot-alias-locale-settings '{"en_US":{"enabled":true}}' \
  --profile PROFILE
```

## Associate with Connect

```bash
aws connect associate-bot \
  --instance-id INSTANCE_ID \
  --lex-v2-bot '{"AliasArn":"arn:aws:lex:REGION:ACCOUNT:bot-alias/BOT_ID/ALIAS_ID"}' \
  --profile PROFILE
```

## Q Connect AI Prompt

```bash
# Write YAML to file first, then:
CONFIG_JSON=$(python3 -c "
import json
with open('/tmp/prompt.yaml') as f:
    yaml_text = f.read()
print(json.dumps({'textFullAIPromptEditTemplateConfiguration': {'text': yaml_text}}))
")

aws qconnect create-ai-prompt \
  --assistant-id ASSISTANT_ID \
  --name "my-prompt" \
  --type "SELF_SERVICE_PRE_PROCESSING" \
  --model-id "anthropic.claude-3-haiku-20240307-v1:0" \
  --template-type "TEXT_COMPLETIONS" \
  --api-format "ANTHROPIC_CLAUDE_TEXT_COMPLETIONS" \
  --template-configuration "$CONFIG_JSON" \
  --visibility-status SAVED \
  --profile PROFILE
```

## Full Deploy Pipeline

```bash
# Build
aws lexv2-models build-bot-locale --bot-id $BOT_ID --bot-version DRAFT --locale-id en_US --profile PROFILE

# Wait for build
while true; do
  STATUS=$(aws lexv2-models describe-bot-locale --bot-id $BOT_ID --bot-version DRAFT --locale-id en_US --profile PROFILE --query 'botLocaleStatus' --output text)
  echo "Status: $STATUS"
  if [ "$STATUS" = "Built" ]; then break; fi
  if [ "$STATUS" = "Failed" ]; then echo "BUILD FAILED"; exit 1; fi
  sleep 5
done

# Create version
aws lexv2-models create-bot-version --bot-id $BOT_ID --bot-version-locale-specification '{"en_US":{"sourceBotVersion":"DRAFT"}}' --profile PROFILE

# Update alias to new version (or create new alias)
aws lexv2-models update-bot-alias --bot-id $BOT_ID --bot-alias-id $ALIAS_ID --bot-alias-name "LiveAlias" --bot-version "$NEW_VERSION" --profile PROFILE
```
