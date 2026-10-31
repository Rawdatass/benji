# Learning Log

Building Benji in public while learning AI engineering. One entry per phase:
what I built, what I learned, what broke and how I fixed it.
(Future me and interviewers: this is the real story.)

---

## Phase 0: Foundations (2026-10-31)

**Built**
- Problem statement, user stories, architecture ("one engine, many doors")
- Repo hygiene: `.gitignore`, `.env.example`, MIT license
- 60-second concept demo video (HTML/CSS animation → Playwright → ffmpeg)

**Learned**
- *Design before code.* Nine questions to answer first: who, what, data, where truth lives, trust, brain vs doors, failure modes, stack, smallest version.
- *Secrets never go in git.* `.env` holds real keys and is ignored. `.env.example` shows which keys exist, with no values, and is committed.
- *Channel risk.* WhatsApp's Business API restricts general-purpose AI chatbots, so the engine must not depend on any one chat app.

**Prior work:** FinanceOS, my personal finance system (PostgreSQL, Docker, an importer, MCP tools, a CFO agent plus 5 sub-agents).
Real bugs from that build: Postgres port not exposed, Python 3.9 too old for MCP, a library renamed in v2,
Streamlit tables rendering at 0 width inside hidden tabs, and transfers wrongly counted as spending.

---

## Phase 1: Engine in the terminal
_(next)_
