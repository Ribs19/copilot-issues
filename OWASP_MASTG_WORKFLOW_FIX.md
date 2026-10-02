# OWASP MASTG Moderator Workflow Fix

## Problem Statement
GitHub Actions job in OWASP/mastg fails with error:
```
##[error]API error: TypeError: Cannot use 'in' operator to search for 'choices' in OK
```

**Job URL:** https://github.com/OWASP/mastg/actions/runs/35989247147/job/107599121514

## Root Cause Analysis

### Error Details
The error occurs in the "Detect spam or low-quality content" step of the moderator workflow (`.github/workflows/moderator.yml`). This step uses `actions/ai-inference@v1` to call the GitHub AI inference API.

### What Causes It
1. The AI inference API endpoint returns an unexpected response
2. Instead of returning a JSON object with a `choices` field, it returns the string "OK"
3. The action's internal code tries to check if the response contains `'choices'`
4. Using the `in` operator on a string "OK" instead of an object causes a TypeError
5. The job fails immediately without error handling

### Why This Matters
- The workflow job terminates with a failure status
- No moderation feedback is generated for PRs
- The repository's moderation system is compromised

## Solution

### Fix: Add Error Handling
Add `continue-on-error: true` to the "Detect spam or low-quality content" step in `.github/workflows/moderator.yml`.

This allows the workflow to continue even if the AI inference call fails. The downstream steps are already designed to handle a missing or empty response:
- "Write audit summary" step checks if `RAW_MODEL_OUTPUT` is set
- "Apply label and comment if needed" step checks if `steps.ai.outputs.response != ''`

### The Change
**File:** `.github/workflows/moderator.yml`

**Location:** The "Detect spam or low-quality content" step (around line 159)

**Before:**
```yaml
- name: Detect spam or low-quality content
  id: ai
  if: steps.allow.outputs.skip != 'true'
  uses: actions/ai-inference@v1
  with:
    model: openai/gpt-4o-mini
    system-prompt: |
      ...
```

**After:**
```yaml
- name: Detect spam or low-quality content
  id: ai
  if: steps.allow.outputs.skip != 'true'
  uses: actions/ai-inference@v1
  continue-on-error: true
  with:
    model: openai/gpt-4o-mini
    system-prompt: |
      ...
```

### Why This Works
- The `continue-on-error: true` keyword tells GitHub Actions to set the job outcome as success even if this step fails
- The downstream steps are defensive and already check if the response exists before using it
- If the API call fails, the workflow completes with status "success with warnings" instead of failing completely
- The moderation system gracefully degrades when the AI service is unavailable

## Implementation Instructions

1. Navigate to the OWASP/mastg repository
2. Edit `.github/workflows/moderator.yml`
3. Find the "Detect spam or low-quality content" step (around line 159)
4. Add `continue-on-error: true` after the `uses: actions/ai-inference@v1` line
5. Commit and push the change
6. The fix will take effect for the next PR that triggers the workflow

## Testing
The fix can be verified by:
1. Creating a new PR in OWASP/mastg that triggers the moderator workflow
2. Observing that the workflow completes successfully even if the AI inference API has issues
3. Checking that the other moderation steps (heuristic signals, change summary) still execute

## Additional Context
- **Workflow file:** `.github/workflows/moderator.yml` in OWASP/mastg
- **Problem:** API response parsing fails when API returns non-JSON response
- **Risk level:** Low - defensive programming with minimal logic changes
- **Breaking changes:** None - only affects error handling behavior
