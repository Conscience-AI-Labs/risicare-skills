# risicare — the Agent Skill for Risicare

One skill that teaches a coding agent to add, extend, debug and explain
[Risicare](https://risicare.ai) tracing in a Python or JavaScript project, and to prove that a trace arrived.

It targets the released SDKs: `risicare>=0.6.0` on PyPI and `risicare@>=0.9.0` on npm.

## Install

**Any agent that reads Agent Skills (Claude Code, Codex, Cursor and others):**

```
npx skills add Conscience-AI-Labs/risicare-skills --skill risicare
```

**Claude Code, as a plugin:**

```
/plugin marketplace add Conscience-AI-Labs/risicare-skills
/plugin install risicare
```

**Codex and Cursor** read the skill from `.agents/skills/risicare/` in your project. `npx skills add` writes it there.

A manual install does not update itself. Run `npx skills update` to get a new version.

## Use

Ask your agent, for example:

- "Add Risicare tracing to this application and confirm a trace arrived."
- "Audit the Risicare instrumentation in this repository."
- "What does this Risicare error code mean?"

You need a project API key from the Risicare dashboard. Set it as `RISICARE_API_KEY` in your environment.
Do not paste it into the chat.

## What is in this repository

| Path | Content |
|---|---|
| `skills/risicare/SKILL.md` | The skill: detection, routing, what an agent can and cannot do, the verify step. |
| `skills/risicare/references/` | One reference per task, per language. The agent reads one when the skill routes to it. |

## Feedback and support

If the skill gave wrong or missing guidance, ask your agent to follow the skill's feedback reference. The agent
drafts the report and shows it to you. After you approve it, you send it to support@risicare.ai with the subject
`Risicare skill feedback: <title>`. The agent does not send email.

An SDK or service defect goes to support@risicare.ai, or to the team's shared Slack channel with Risicare if the
team has one.

## Licence

Proprietary: the Risicare Skill License of Conscience Labs Pvt. Ltd. `LICENSE` has the complete terms. With an
active Risicare subscription (the private beta included) you may install the skill, keep the installed copy in your
project's repository, and use its code examples in your applications. The copy in `skills/risicare/LICENSE` installs
with the skill. The Risicare SDKs keep their own licence.
