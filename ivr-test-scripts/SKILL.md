---
name: ivr-test-scripts
description: Generate exhaustive test scripts for Amazon Connect IVR flows and Lex bots covering all call paths, error conditions, and edge cases. Trigger on test scripts, test scenarios, IVR testing, call flow testing, or QA scripts.
---

# IVR Test Script Generation Skill

Generate exhaustive, path-complete test scripts for Amazon Connect IVR flows and Lex bots.

## Quick Reference

| Task | Guide |
|------|-------|
| Analyze flow and enumerate paths | [analyze-flow.md](analyze-flow.md) |
| Generate test scripts | [generate-scripts.md](generate-scripts.md) |
| QA the test scripts | [qa-scripts.md](qa-scripts.md) |
| AWS Native Test API reference | [native-test-api.md](native-test-api.md) |

## Core Principles

1. **100% path coverage** — every path from start to terminal node gets at least one scenario.
2. **Concise format** — `System:` / `Caller:` lines only. No `Purpose:` blocks.
3. **Input method matching** — check the Lex bot configuration to determine whether voice, DTMF, or both are supported, and test only the corresponding method(s). Do not assume both are always available.
4. **Error paths are first-class** — timeout, invalid input, unrecognized speech get dedicated scenarios. Each scenario must indicate whether it is testable via the text chat API (e.g., timeout scenarios are not testable via text chat).
5. **Exact system prompts** — `System:` lines quote verbatim TTS text from the flow JSON.
6. **Test conditions required** — each scenario states when it can be tested (e.g., Mon-Fri, Sat-Sun).
7. **QIC prompt coverage** — when the flow uses Amazon Q in Connect (QIC), locate the associated AI prompt and understand its instructions, persona, and knowledge scope. Then generate diverse test scenarios with various valid caller inputs that exercise different parts of the prompt (e.g., different intents, edge-case phrasing, topics the AI should and should not handle).

## Prerequisite Questions

**IMPORTANT: All questions to the user MUST be presented as selection menus (not free-text). Use the `mcp_question` tool with predefined options. Only add a custom/free-text option when no finite set of choices exists.**

Before starting, present these as selection questions:

1. **Input source** (selection):
   - Upload JSON
   - Pull from AWS
   - Describe a new design
2. **Output delivery** (selection):
   - Save to AWS (S3)
   - Export as local files
3. **AWS profile** (selection): Run `aws configure list-profiles` to discover available profiles and present them as options.

If pulling from AWS:
- Run `aws sso login --profile <PROFILE>` if needed
- Run `aws connect list-instances --profile <PROFILE>` to get instances
- Present instances as a selection menu for the user to pick from
- Present matching flows as a selection menu if multiple results match

**Confirmation gate:** Summarize back: "I will generate test scripts for [flow name] from [source]. Correct?" Do NOT proceed until confirmed.

After confirmation, also collect via selection where possible:
- Phone number to call (if assigned — present discovered numbers as options)
- Special instructions (e.g., "focus on error paths" — this may require free-text)

**CRITICAL:** Confirm profile and instance before any AWS commands. Wrong instance = wrong flow.

## Workflow

1. Collect inputs and confirm with user (see Prerequisite Questions)
2. Read [analyze-flow.md](analyze-flow.md) — parse flow JSON, walk all paths, enumerate every unique route
3. Read [generate-scripts.md](generate-scripts.md) — produce test scripts in the required format
4. Read [qa-scripts.md](qa-scripts.md) — verify completeness, find gaps, add missing scenarios
5. If creating programmatic tests, read [native-test-api.md](native-test-api.md)
6. Export to file

## What To Do

- Parse the flow JSON first — walk the entire graph from `StartAction` to every terminal node
- Extract exact prompts — copy the `Text` parameter verbatim into `System:` lines
- Group by scenario type — happy paths first, then errors, timeouts, edge cases
- Include edge cases — wrong keys, silence, special keys, hesitant speech, out-of-domain phrases
- Number scenarios sequentially — `S1`, `S2`, etc. with descriptive names

## What To Avoid

- Do NOT use tables for dialogue — use `Caller:` / `System:` format only
- Do NOT write robotic caller lines — BAD: `Caller: "1"` GOOD: `Caller: Press 1`
- Do NOT paraphrase system prompts — `System:` text must match flow JSON exactly
- Do NOT skip error paths — errors at EVERY menu level must be tested
- Do NOT add verbose descriptions — no `Purpose:` blocks; scenario name and test condition suffice
- Do NOT create duplicate scenarios — each path combination appears exactly once
- Do NOT include Coverage Matrix, Bot Utterance Reference, or appendix sections in output

## Output Format

Single markdown file containing ONLY metadata header and scenario scripts:

```markdown
# [Flow Name] - Test Scripts

**Flow ID:** [flow-id]
**Instance ID:** [instance-id]
**Bots:** [bot names]

---

## S1: [Descriptive Name]
**Test when:** [condition]
**Test API:** Testable / Not testable via Test API — [reason if not testable]

System: "[exact prompt text]"
Caller: [action]
System: "[response text]"
Caller: [next action]
System: Transfers to [queue] / Call disconnects

---

## S2: [Name]
...
```
