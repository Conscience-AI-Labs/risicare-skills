---
name: skill-feedback
description: "Read when the user says this skill gave wrong or missing guidance."
---
# Skill feedback

Use this when the user says that this skill gave wrong, missing or outdated guidance. A defect
in the Risicare SDK or service is not skill feedback. Tell the user so. An SDK or service
defect goes to support@risicare.ai, or to the team's shared Slack channel with Risicare if the
team has one.

## Steps

1. Draft the report. Do not send or file anything.
2. Before you show the draft, remove secrets, keys, prompt and completion text, and customer
   data from it (see "What the report never contains"). Tell the user in one line that you
   removed them.
3. Show the full draft to the user. Ask for approval. Change the draft if the user asks.
4. After the user approves, give the user the final text and the address
   `support@risicare.ai`, with the subject `Risicare skill feedback: <title>`. Also give a
   `mailto:` link with the subject filled:
   `mailto:support@risicare.ai?subject=Risicare%20skill%20feedback%3A%20<title>`. In the link,
   write each space of the title as `%20`, and leave out `&`, `#` and `?`. The user sends it.
5. Do not send the email yourself. Do not open an issue: the skill has no public issue tracker.

## What the report contains

- Title: one line, the wrong or missing guidance.
- The file and section of this skill (for example `SKILL.md` Hard rules, or the Python
  setup reference, Steps).
- What the skill said, quoted.
- What happened instead, and how you know (the command, the WARNING text, the metric values).
- What the guidance should say, if known.
- Environment: language, SDK version from package metadata, framework and provider versions,
  the agent harness, and the skill `metadata.version`.

## What the report never contains

- API keys, tokens or any other secret, also not partly masked.
- Prompt or completion text, trace content, user data or customer names.
- The project UUID, unless the user asks to include it.
- Private source code beyond the few lines that show the problem, and only with approval.
