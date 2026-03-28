# Prompt Template Format

Use this structure for all prompt templates in `./ai-lab/prompts/`.

---

## Standard Structure

```markdown
# [Task Name]

## Role
You are a DevOps engineer assistant. Your job is to [specific task].

## Instructions
1. [Step 1]
2. [Step 2]
3. [Step 3]

## Output Format
[Specify the exact format — JSON, bullet list, table, checklist, etc.]

## Input
{{INPUT_PLACEHOLDER}}
```

---

## Placeholder Conventions

| Placeholder | Meaning |
|-------------|---------|
| `{{INPUT_LOG}}` | Raw log file content |
| `{{INPUT_INCIDENT}}` | Incident description or summary |
| `{{INPUT_SERVICE}}` | Service name |
| `{{INPUT_CONTEXT}}` | Additional background context |
| `{{DATE}}` | Current date (injected at runtime) |

---

## Example: Log Summary Template

```markdown
# Log Summary Request

## Role
You are a DevOps engineer. Analyze the log below and produce a structured summary.

## Instructions
1. Count total errors, warnings, and info messages
2. Identify the root cause of the most critical issue
3. Note when the incident started and ended
4. List recommended next steps

## Output Format
Use this exact structure:
- **Error count:** [number]
- **Warning count:** [number]
- **Root cause:** [one sentence]
- **Timeline:** [start] to [end]
- **Next steps:** [numbered list]

## Input Log
{{INPUT_LOG}}
```

---

## Tips

- Always specify the output format — vague prompts produce vague answers
- Keep instructions numbered and unambiguous
- Test with `./ai-lab/samples/sample-errors.log` before using other data
- Reference `00-style.md` in the prompt if you want consistent formatting across all outputs
