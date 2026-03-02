# QA Test Scripts

**Assume there are gaps. Your job is to find them.**

## Phase 1: Path Coverage

### 1. Functional Path Completeness
- [ ] Count all unique end-to-end paths in the IVR flow (each intent, each branch)
- [ ] Verify each functional path has at least one scenario
- [ ] Verify time-dependent paths (open/closed hours) each have a scenario
- [ ] Verify each menu level has one error/timeout scenario

### 2. Prompt Accuracy
- [ ] For every SYSTEM line, verify the text matches the flow JSON `Parameters.Text` EXACTLY
- [ ] No paraphrasing, no missing sentences, no added words
- [ ] Check for copy-paste errors (wrong prompt in wrong scenario)

### 3. Input Accuracy
- [ ] DTMF digits match the flow JSON `Conditions.Operands` values
- [ ] Voice utterances (if used) exist in the Lex bot's sample utterances
- [ ] Intent names in the analysis match actual Lex bot intent names

### 4. Scenario Uniqueness
- [ ] No two scenarios have identical paths
- [ ] No duplicate scenario IDs
- [ ] Each scenario covers a distinct path or condition

## Phase 2: Trigger Condition Verification

### 1. Every Scenario Has a Trigger
- [ ] Every scenario has a "**Triggered when:**" line
- [ ] Trigger conditions are specific and complete (not vague like "when caller calls")
- [ ] Time-dependent triggers specify exact hours/days
- [ ] Multi-step triggers describe all preceding steps

### 2. Error Path Coverage (Lightweight)
For each menu level (each `ConnectParticipantWithLexBot` or `GetParticipantInput` block):
- [ ] At least one error scenario exists
- [ ] Error outcome matches the flow JSON error transitions
- [ ] Error prompt text matches the flow JSON

## Phase 3: Coverage Matrix

- [ ] Coverage matrix lists every path through the IVR
- [ ] Every path has at least one scenario number assigned
- [ ] No paths are missing from the matrix

## Verification Loop

1. Generate all test scripts
2. Run Phase 1 path coverage checks
3. Run Phase 2 trigger condition checks
4. Run Phase 3 coverage matrix check
5. **List all gaps found**
6. Add missing scenarios
7. **Re-verify** — adding scenarios may reveal more gaps
8. Repeat until all checks pass

**Do not declare complete until:**
- Every functional path has a scenario
- Every scenario has a trigger condition
- All prompt text is verified verbatim against flow JSON
- Coverage matrix accounts for all paths
