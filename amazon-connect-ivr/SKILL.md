---
name: amazon-connect-ivr
description: Create, edit, validate, and deploy Amazon Connect contact flow (IVR) JSON. Trigger on IVR, contact flow, call flow, phone tree, flow JSON, DTMF menu, or Connect deployment references.
---

# Amazon Connect IVR Skill

Generate and manage Amazon Connect contact flow JSON (Version 2019-10-30) for voice IVR systems.

## Quick Reference

| Task | Guide |
|------|-------|
| Create IVR from scratch | [create-from-scratch.md](create-from-scratch.md) |
| Edit existing IVR JSON | [edit-existing.md](edit-existing.md) |
| QA and validate flow | [qa-validation.md](qa-validation.md) |
| Component reference (25 block types) | [flow-components.md](flow-components.md) |
| Deployment bugs and test API rules | [deployment-gotchas.md](deployment-gotchas.md) |

## Core Principles

1. Use the minimum blocks needed. Do not over-engineer.
2. Every branch must terminate at `DisconnectParticipant` or a transfer block.
3. Every block that can error must have error transitions handled.
4. Every action must have a unique v4 UUID as its `Identifier`.
5. Output must be valid Amazon Connect flow JSON (Version `2019-10-30`).
6. Include `Metadata` with positions so flows render in the Connect console.
7. Confirm requirements with the user before generating any JSON.

## Correct Block Type Names

| WRONG (deploy fails) | CORRECT |
|---|---|
| `InvokeExternalResource` | `InvokeLambdaFunction` |
| `Loop` | `Compare` + `UpdateContactAttributes` counter |

Common parameter mistakes:
- `FunctionArn` is wrong -- use `LambdaFunctionARN`
- `TimeLimit` is wrong -- use `InvocationTimeLimitSeconds`
- Omitting `TextToSpeechStyle: "None"` from `UpdateContactTextToSpeechVoice`

Required block ordering:
- `UpdateContactTargetQueue` BEFORE `TransferContactToQueue` (queue must be set first)
- `UpdateContactTargetQueue` BEFORE `CheckHoursOfOperation` (hours check uses queue's hours; no queue = always closed)
- `CreateWisdomSession` BEFORE `ConnectParticipantWithLexBot` with `AMAZON.QinConnectIntent`

See [flow-components.md](flow-components.md) for all 25 verified block types with schemas.

## What To Do

- Confirm prompts, inputs, branches, and error handling before generating JSON.
- Use `Text` parameter in `MessageParticipant` for TTS (unless user specifies audio files).
- Integrate a Lex V2 bot (via `amazon-lex-bot` skill) when callers need to "press or say" options.
- Include `Metadata` with `entryPointPosition` and `ActionMetadata` positions for every block.
- Space blocks ~250px horizontal, ~150px vertical between branches.
- Save final flow JSON to a file the user can import.

## What To Avoid

- `InvokeExternalResource` or `Loop` action types -- both are rejected by the API.
- `GetParticipantInput` for multi-digit input -- it only supports single digits (0-9, #, *). Use `StoreInput: "True"` for multi-digit.
- Hardcoded contact flow ARNs -- use placeholder comments.
- Empty error branches -- always route errors to a prompt then disconnect or retry.
- Duplicate UUIDs across blocks -- this corrupts the flow.
- Omitting `Version`, `Transitions: {}` on terminal blocks, or `Errors` array on non-terminal blocks.
- Missing `Text` parameter inside `GetParticipantInput` -- the prompt belongs in the block, not a separate `MessageParticipant`.
- Mixing up `GetParticipantInput` (DTMF-only) and `ConnectParticipantWithLexBot` (Lex V2 / Nova Sonic).
- Using `ConnectParticipantWithLexBot` without a deployed Lex bot alias ARN.
- Using `AMAZON.QinConnectIntent` without calling `CreateWisdomSession` first.

See [deployment-gotchas.md](deployment-gotchas.md) for additional deployment bugs and test API compatibility rules.

## Flow JSON Structure

```json
{
  "Version": "2019-10-30",
  "StartAction": "<UUID of first action>",
  "Metadata": {
    "entryPointPosition": { "x": 39, "y": 40 },
    "ActionMetadata": {
      "<action-uuid>": {
        "position": { "x": 250, "y": 200 }
      }
    }
  },
  "Actions": [
    {
      "Identifier": "<UUID>",
      "Type": "<ActionType>",
      "Parameters": { },
      "Transitions": {
        "NextAction": "<UUID>",
        "Conditions": [ ],
        "Errors": [ ]
      }
    }
  ]
}
```

## Prerequisite Questions

Before starting ANY work, ask ALL these questions together in a single `mcp_question` call:

1. **Input source** -- Upload existing JSON, pull from AWS, or create new?
2. **Output delivery** -- Deploy directly to AWS, or export as JSON files only?
3. **AWS profile** -- Which AWS CLI profile to use?

If AWS deployment is chosen:
1. Run `aws sso login --profile <PROFILE>` if needed.
2. Run `aws connect list-instances --profile <PROFILE>` to get instances.
3. Ask user to select instance from the list.

After collecting answers, confirm back to the user:
> "I will [create/edit] the IVR flow and [deploy to AWS profile `X`, instance `Y` / save as JSON files]. Is that correct?"

Do NOT proceed until the user confirms. Deploying to the wrong Connect instance can cause production outages.

## Workflow

1. Ask prerequisite questions -- wait for user confirmation.
2. Gather detailed IVR requirements.
3. If Lex or Nova Sonic integration needed, use the `amazon-lex-bot` skill first.
4. Read [create-from-scratch.md](create-from-scratch.md) or [edit-existing.md](edit-existing.md).
5. Generate the flow JSON.
6. Run QA per [qa-validation.md](qa-validation.md) -- loop until all issues resolved.
7. Export the final JSON file.
8. If AWS deployment: deploy using the commands below.

## Deploying to AWS

### Create new flow:
```bash
aws connect create-contact-flow \
  --instance-id <INSTANCE_ID> \
  --name "<Flow Name>" \
  --type CONTACT_FLOW \
  --content "$(cat flow.json)" \
  --profile <PROFILE>
```

### Update existing flow:
```bash
aws connect update-contact-flow-content \
  --instance-id <INSTANCE_ID> \
  --contact-flow-id <FLOW_ID> \
  --content "$(cat flow.json)" \
  --profile <PROFILE>
```

### Verify deployment:
```bash
aws connect describe-contact-flow \
  --instance-id <INSTANCE_ID> \
  --contact-flow-id <FLOW_ID> \
  --profile <PROFILE>
```