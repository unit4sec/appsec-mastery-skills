# AppSec Mastery — Implementation Skills

Ready-to-use **AI implementation skills** for the *Advanced Web & Mobile Application Security* course.

Each course topic ships one skill: a structured prompt you hand to your coding AI
(Claude Code, Cursor, Copilot, etc.) so it implements that security control
**correctly and securely** in *your* stack. The course teaches you the theory —
what the control is, the risks, and why each defense matters — and these skills let
you act on it without hand-rolling the dangerous parts.

## How to use

1. Open the topic folder (e.g. `oauth-security/`).
2. Copy the contents of `SKILL.md` into your AI coding assistant:
   - **Claude Code / Agent skills:** drop the folder into `.claude/skills/`.
   - **Cursor:** paste into a project rule, or `.cursor/rules`.
   - **Any chat-based AI:** paste `SKILL.md` as the system/first message, then describe your app.
3. Tell it your framework/language and let it implement — then review against the checklist in the skill.

> These skills encode current best practice (relevant RFCs, CWE mitigations, OWASP guidance).
> They favor **configuring a trusted provider and using maintained libraries** over custom crypto/auth code.

## Topics

| Topic | Skill | Course lectures |
|-------|-------|-----------------|
| OAuth Security | [`oauth-security/SKILL.md`](oauth-security/SKILL.md) | OAuth Parts 1–3 |
| Strong Customer Authentication | [`strong-customer-authentication/SKILL.md`](strong-customer-authentication/SKILL.md) | SCA Parts 1–2 |

_More topics are added lecture by lecture as the course grows._
