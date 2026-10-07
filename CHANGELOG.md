# Changelog

Semantic versioning: patch for clarifications and fixes, minor for new capability, major for removed or
renamed behaviour.

## 0.1.1 (unreleased)

Moves the skill to the released SDKs: `risicare>=0.6.0` (PyPI) and `risicare@>=0.9.0` (npm). The two are minor releases, so
`~=0.5.0` and `^0.8.0` do not take them.

- A true `flush()` now means that every accepted span was delivered, also in a signal handler and after `shutdown()`. A
  loss makes one `flush()` false, and the next one true. The Verify steps say so. JavaScript: register your own SIGTERM or SIGINT
  listener before `init()` if it makes spans that must be delivered.
- With no API key the SDK sends nothing, `is_enabled()` / `isEnabled()` and `flush()` are false, and `init()` logs one
  WARNING. `flush()` is also false before `init()` and with tracing off.
- A score outside [0, 1] counts as a failed score: the next `flush()` is false.
- Fixed on every version: the failed-exports count counts each export call that failed as a whole; a later round can still deliver. A 429
  that a retry in the same call repairs counts 0.
- Older SDKs: `references/older-versions.md` has a new band for Python 0.5.0 to 0.5.1 and npm 0.8.0. It lists what is different on
  that band, what changes on the upgrade, and how to upgrade across the minor version.
- Not measured by agent runs: every text change of this release, and the licence text. The checks were loopback probes of each
  changed sentence against the published SDKs, and the replay of the recorded test runs on the new SDK versions.

## 0.1.0 (unreleased)

First version of the `risicare` Agent Skill.

- What it does: it adds Risicare tracing to one Python or JavaScript/TypeScript service and proves
  that the spans arrive. It covers setup, agent and session identity, think/decide/act phases,
  multi-agent messages, scores, error codes, older-SDK upgrades and troubleshooting. It tells the
  user which features are not available to customers, and it does not build against them. It never
  asks for the API key and never prints it.
- SDK floor: `risicare>=0.5.1` (PyPI) and `risicare@>=0.8.0` (npm).
- Measured in Claude Code, headless, against a loopback gateway, 3 runs per case. Each pair is
  without the skill → with the skill. The measured text is this release before three late edits:
  the licence value and text, the support path and the feedback-by-email flow. Agent runs did not measure
  those three edits.
  - Sonnet: pass rate outcome 84.8 % → 97.0 %, fault 76.2 % → 95.2 %, refusal 60.0 % → 93.3 %. Runs that showed the key: 16 → 0. Unavailable features in the code: 1 → 0; calls to closed routes: 6 → 0. Triggers: 12 of 12 cases; false triggers: 0 of 10 cases.
  - Opus: pass rate outcome 93.9 % → 100 %, fault 76.2 % → 95.2 %, refusal 53.3 % → 93.3 %. Runs that showed the key: 12 → 2, both in runs where the skill did not load. Unavailable features in the code: 2 → 0; calls to closed routes: 5 → 0. Triggers: 12 of 12 cases; false triggers: 0 of 10 cases.
- Last fixes, from an earlier measurement: list `.env` names only, and never show its values through a
  mask. JavaScript Instructor: patch the OpenAI client. When the user asks for an upgrade, do it
  and report the licence change. Read a reference from the installed folder by its path.
- Licence: proprietary, the Risicare Skill License of Conscience Labs Pvt. Ltd. `LICENSE` has the complete
  terms: install the skill, keep the copy in your project's repository, use its code examples in your
  applications. A copy, `skills/risicare/LICENSE`, installs with the skill.
- Support: an SDK or service defect goes to support@risicare.ai, or to the team's shared Slack channel
  with Risicare if the team has one.
- Skill feedback: the agent drafts the report, removes secrets, keys, prompt and completion text, and
  customer data, and shows it to you. After you approve it, the agent gives you the final text. You send
  it to support@risicare.ai with the subject `Risicare skill feedback: <title>` (the agent also gives a
  `mailto:` link). The agent sends no email and opens no issue.
