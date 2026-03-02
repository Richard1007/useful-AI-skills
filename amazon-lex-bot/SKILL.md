---
name: amazon-lex-bot
description: Create, build, version, and deploy Amazon Lex V2 bots for Connect IVR voice and DTMF input. Trigger on Lex bot, intents, utterances, NLU, or voice recognition references.
---

# Amazon Lex V2 Bot Skill

Create and manage Lex V2 bots for Amazon Connect IVR flows, supporting both traditional NLU and Nova Sonic speech-to-speech.

## Quick Reference

| Task | Guide |
|------|-------|
| Create bot from scratch | [create-bot.md](create-bot.md) |
| Nova Sonic S2S bot | [create-bot.md](create-bot.md) → Nova Sonic section |
| CLI command reference | [quick-commands.md](quick-commands.md) |
| QA and validate bot | [qa-validation.md](qa-validation.md) |
| Intent/utterance reference | [intent-reference.md](intent-reference.md) |

## Core Principles

1. **Diverse utterances** — 8-12 per intent covering formal, informal, synonyms, and DTMF digits.
2. **Build before deploy** — `build-bot-locale` is async; poll until `Built` before creating version or alias.
3. **Wait after version creation** — Bot version is async; wait 5-10s before `create-bot-alias`.
4. **Test before connecting** — Validate bot responses before referencing in a Connect flow.
5. **Use AmazonConnect-specific Lex service role** for Q Connect bots (`AWSServiceRoleForLexV2Bots_AmazonConnect_*`). Generic role causes `AccessDenied` on `wisdom:SendMessage`.
6. **Version Q Connect prompts** — Call `create-ai-prompt-version` before referencing in an AI agent.
7. **Nova Sonic: us-east-1 and us-west-2 only** — Requires generative voice engine (`"engine":"generative"`).

## Prerequisites

Before starting, confirm with the user:
1. **Input source** — Upload JSON, pull from AWS, or create new?
2. **Output delivery** — Deploy to AWS or export JSON files?
3. **AWS profile** — Which CLI profile?

If deploying: run `aws sso login`, then `aws connect list-instances` to select the target instance.

**Confirmation gate:** Summarize bot name, action, profile, and Connect instance. Do NOT proceed until confirmed.

## What To Do

- One bot per IVR menu level — avoid DTMF digit conflicts across menus
- Set `nluIntentConfidenceThreshold` between 0.40 (lenient) and 0.80 (strict)
- Include `--bot-alias-locale-settings '{"en_US":{"enabled":true}}'` when creating aliases
- Add `AMAZON.QinConnectIntent` for Q Connect AI agent handoff
- Write Q Connect prompts to file — use `file://` to avoid JSON escaping issues
- Associate Q Connect assistant with Connect instance before `CreateWisdomSession`
- Configure DTMF-to-intent mapping for both keypad and voice input

## What To Avoid

- Overlapping utterances across intents — causes mismatches
- Skipping FallbackIntent configuration — every bot needs it
- Hardcoding bot IDs — always capture from create command output
- Deleting a bot referenced by a live flow
- Reusing DTMF digits across intents in the same bot
- Using `SELF_SERVICE_ANSWER_GENERATION` type — use `SELF_SERVICE_PRE_PROCESSING` with `ANTHROPIC_CLAUDE_TEXT_COMPLETIONS`
- Creating alias without locale settings — alias will reject all requests

## Bot Lifecycle

### Traditional Lex Bot
```
1. create-bot → botId
2. create-bot-locale (DRAFT, en_US) → language, voice, threshold
3. create-intent (each) → utterances, slots
4. build-bot-locale → async, poll until Built
5. create-bot-version → immutable version (1, 2, 3...)
6. create-bot-alias → aliasId, aliasArn
7. associate-bot → available in Connect flows
```

### Nova Sonic S2S Bot (with Q Connect)
```
1. create-bot → botId (poll until Available)
2. create-bot-locale WITH voice-settings (engine: generative)
3. create-intent (routing + AMAZON.QinConnectIntent)
4. build-bot-locale → async, poll until Built
5. create-bot-version → immutable version
6. create-bot-alias → aliasArn
7. associate-bot → available in Connect flows
8. create-ai-prompt (SELF_SERVICE_PRE_PROCESSING, YAML)
9. create-ai-prompt-version → required before agent use
10. create-ai-agent (SELF_SERVICE, references prompt version)
11. create-ai-agent-version
12. update-assistant-ai-agent → set as default SELF_SERVICE agent
```

## Workflow

1. Gather requirements: bot name, intents, utterances, DTMF mappings, locale, threshold
2. Check existing resources: `list-bots`, IAM roles
3. Create bot and capture botId
4. Create locale (add `--voice-settings` for Nova Sonic)
5. Create each intent with 8-12 diverse utterances
6. Build bot locale and poll until `Built`
7. Create bot version (wait 5-10s for async completion)
8. Create alias with locale settings enabled
9. Associate with Connect instance
10. Run QA validation — see [qa-validation.md](qa-validation.md)
11. (Nova Sonic) Set up Q Connect AI prompt, agent, and assistant

## Native Test API

DTMF digits configured as utterances are correctly matched when Connect's Native Test API sends `DtmfInput` actions. Lex bots are fully compatible with the test framework.
