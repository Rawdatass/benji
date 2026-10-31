# Benji — Architecture & Build Plan

> A personal finance assistant you text like a friend: on WhatsApp, inside Claude, or on the web.
> Written for learning: every decision explains **why**, not just **what**.

---

## 0. How to think about any system (use this every time)

Before writing code, answer these 9 questions in order. Most bad projects skip straight to #8.

| # | Question | For this project |
|---|---|---|
| 1 | **Who** has the problem? | People who use AI daily but never open a budgeting app |
| 2 | **What** do they need to do? (user stories) | Log a purchase in 5 seconds; ask "can I afford this?" |
| 3 | **What facts** must we remember? (data) | Users, transactions, income, bills, goals |
| 4 | **Where does the truth live?** | One database. Never in the AI's memory |
| 5 | **Who is allowed to do what?** (trust) | Each user sees only their own data; AI can only use our tools |
| 6 | **What is the brain vs the doors?** | One engine; WhatsApp / Claude / web are just doors |
| 7 | **What can go wrong?** | Wrong numbers, leaked data, spam, AI costs, platform bans |
| 8 | **What do we build it with?** | Tech stack (section 5) |
| 9 | **What is the smallest version that proves it works?** | MVP (section 7, Phase 1–2) |

---

## 1. Problem statement

People track their money badly, not because they don't care, but because **tracking is friction**:
opening an app, finding the category, typing the amount. So they stop after a week.
And when they wonder *"can I afford this?"*, they guess, because their finances are spread
across bank apps, cards and their head.

> **Problem:** There's no zero-effort way to record spending and get an honest
> "can I afford this?" answer, in the chat apps people already use every day.

## 2. Solution

**Benji**: a finance assistant that lives in the chat apps you already use.

| You type | It does |
|---|---|
| "Bought Call of Duty at the mall, $100" | Saves it: $100, Entertainment, today. Replies "Saved ✅ (reply UNDO to remove)" |
| "Can I buy a $100 game?" | Checks income, bills, goals and this month's spending, then says *"Better wait. You have $64 left to spend safely until payday on Nov 14."* |
| "How much did I spend on food this month?" | Answers from the database |
| *sends a photo of a receipt or a bank CSV* | Extracts the transactions, shows a preview, saves when you say yes |
| (Sunday evening, automatically) | Sends a short weekly summary |

Same assistant, same data, **three doors**: WhatsApp, Claude (via MCP), and a web dashboard.

### What it is NOT (say this clearly to users and on LinkedIn)
- Not a bank. It never moves money and never asks for bank passwords.
- Not a licensed financial advisor. It does budgeting and spending guidance, not investment advice.

---

## 3. The big idea: one engine, many doors

This is the most important architecture lesson in this document.

```
   DOORS (channels)                 ENGINE (the brain)                    TRUTH
 ┌──────────────────┐        ┌──────────────────────────────┐      ┌──────────────┐
 │ WhatsApp         │──┐     │  1. Who is this user?        │      │              │
 ├──────────────────┤  │     │  2. AI agent (Claude)        │      │  PostgreSQL  │
 │ Telegram         │──┼────►│     decides which TOOL to use│─────►│  (per-user   │
 ├──────────────────┤  │     │  3. TOOLS (plain Python):    │      │   data)      │
 │ Claude (MCP)     │──┤     │     log_expense, undo,       │◄─────│              │
 ├──────────────────┤  │     │     safe_to_spend, summary…  │      └──────────────┘
 │ Web app          │──┘     │  4. Reply text               │
 └──────────────────┘        └──────────────────────────────┘
```

**Why?** If WhatsApp changes its rules tomorrow (see Risks), you plug in a new door,
like Telegram, and the brain, data and tools stay the same. Each door is about 100 lines of code.
You've already done this once: the FinanceOS MCP tools are reused by Claude Code, the
scheduled reviews and the app page.

### The 3 golden rules (carried over from FinanceOS)
1. **The database is the truth.** The AI never "remembers" balances; it asks the tools every time.
2. **Code calculates, AI explains.** "Can I afford it?" is a formula in Python. The AI only puts it into words.
3. **The AI can only use our tools.** It can't run SQL, can't see other users, and can't do anything we didn't build a tool for.

---

## 4. How the two main flows work, step by step

### Flow A: "Bought Call of Duty for $100"
```
1. You send the message on WhatsApp.
2. Meta's servers POST it to our webhook:  https://api.yourdomain.com/whatsapp
3. Webhook checks the signature (proves it really came from Meta, not a hacker).
4. Find the user by phone number. Unknown number → onboarding message, stop.
5. Rate limit check (e.g. max 50 messages/day per user → protects your AI bill).
6. Engine sends to Claude: system prompt + recent chat + the message + tool list.
7. Claude replies: "call tool log_expense(amount=100, merchant='Call of Duty', category='Entertainment')".
8. Our Python tool validates (amount > 0, sane category), INSERTs with user_id = this user.
9. Tool result goes back to Claude → Claude writes: "Saved ✅ $100 · Entertainment. Reply UNDO to remove."
10. We send that reply through the WhatsApp API. Total time: about 2–4 seconds.
```
**Design choice: save immediately and offer UNDO, instead of asking "Are you sure?"**
For one small expense, a confirmation step is annoying, and annoying = users quit.
UNDO marks the row as voided (it's never deleted; that's the audit trail). For bulk imports
(a CSV of 80 rows), we do preview → confirm, like FinanceOS does today. *Match friction to risk.*

### Flow B: "Can I buy a $100 game?"
```
1-6. Same as above.
7.  Claude calls tool  safe_to_spend()
8.  Python calculates (deterministic, testable, no AI):
       safe_to_spend = income expected this period
                     − fixed bills still due this period
                     − planned goal savings this period
                     − already spent this period
                     − safety buffer (e.g. 10%)
9.  Tool returns: {"safe_to_spend": 64.00, "days_to_payday": 14, "biggest_category": "Dining"}
10. Claude explains: "I'd wait. You have $64 safely left for the next 14 days, and the game is $100.
    If you skip eating out twice this week, you'll have room after payday on Nov 14."
```
**Why not let the AI decide?** AI is great at language and bad at arithmetic it can't check.
A formula can be unit-tested; a vibe cannot.

---

## 5. Tech stack (and why each piece)

| Layer | Choice | Why this one | Cost |
|---|---|---|---|
| Language | **Python 3.12+** | You already know it from FinanceOS; best AI libraries | Free |
| Web framework | **FastAPI** | Small, fast, auto-generated API docs, perfect for webhooks | Free |
| Database | **PostgreSQL** (Docker locally, **Supabase free tier** or the VPS in production) | Same SQL you learned; Supabase adds login + row-level security for free | Free |
| AI | **Claude API** with tool use. **Haiku** for quick logging, **Sonnet** for advice | Cheap model for simple jobs, smart model only when needed | Pay per use (set a hard spend limit) |
| WhatsApp | **Meta WhatsApp Cloud API** (official) | Free test number; replying within 24 hours of a user's message is free | Free for testing |
| Backup channel | **Telegram Bot API** | 100% free, bot-friendly, 15-minute setup | Free |
| Claude door | **MCP server** (you already built two) | Users who live in Claude connect it as a connector | Free |
| Web dashboard | Streamlit now → Next.js on **Vercel free** later | Start simple, upgrade when there are real users | Free |
| Hosting | **One small VPS** (Hetzner / DigitalOcean) running Docker Compose + **Caddy** for HTTPS | Always on (webhooks need this), same Docker skills, one fixed price | ~$5–6/month |
| Local testing | **Cloudflare Tunnel** | Gives your Mac a public HTTPS URL so Meta can reach it while you develop | Free |
| Code & CI | **GitHub** + **GitHub Actions** (runs tests on every push) | Portfolio + safety net | Free |
| Error alerts | **Sentry** free tier | Know when something breaks before users tell you | Free |
| Automation | **n8n**: *optional, not in v1* | Plain Python is clearer for the core loop. Add n8n only if it saves real work | Free |

**Rule for beginners:** every tool you add is one more thing that can break.
Add a tool only when you can say what problem it solves *today*.

### Budget: about $50 for a 3-month trial
| Item | Estimate |
|---|---|
| VPS (3 months) | ~$18 |
| Claude API (set a monthly limit in the Anthropic console) | ~$15–20 |
| Domain name (optional, for a professional URL) | ~$12/year |
| WhatsApp test number, Telegram, Supabase, GitHub, Sentry, Cloudflare | $0 |
| **Total** | **~$45–50** |

Weeks 1–4 cost **$0**, because you develop on your Mac with a Cloudflare Tunnel.

---

## 6. Data design (multi-user)

FinanceOS today is built for **one person**. A product has **many people**, so the key
change is: **every row belongs to a user**.

```sql
users            (id, phone_e164 UNIQUE, display_name, currency, timezone,
                  payday_rule, created_at, deleted_at)
transactions     (id, user_id → users, date, amount NUMERIC, merchant, category,
                  kind, source ['chat','csv','receipt'], voided BOOLEAN, created_at)
recurring_bills  (id, user_id, name, amount, day_of_month)
income_sources   (id, user_id, name, amount, schedule)
goals            (id, user_id, name, target, monthly_contribution, deadline)
messages         (id, user_id, channel, direction, text, created_at)  -- kept 90 days only
change_log       (id, user_id, tool, details JSONB, created_at)      -- audit trail
```

**How we stop user A from seeing user B's data (defense in depth again):**
1. Every tool gets `user_id` **from the engine, never from the AI**. The AI can't ask for another user.
2. Every query has `WHERE user_id = :user_id`.
3. **Row-Level Security** in Postgres. The database itself refuses rows that aren't yours,
   even if the code has a bug.

---

## 7. Build plan: phases, what you'll learn, and "done when"

### Phase 0: Foundations (1–2 days)
- New GitHub repo `benji`, with README, `.gitignore`, `.env.example` and this document.
- Write the problem statement and 5 user stories in the README.
- **Learn:** Git basics, why secrets never go in Git.
- **Done when:** the repo is public with no secrets in it.

### Phase 1: The engine on your Mac, no WhatsApp yet (1 week)
- Multi-user schema (section 6) in Docker Postgres.
- Tools as plain Python functions: `log_expense`, `undo_last`, `safe_to_spend`,
  `spending_summary`, `set_income`, `add_bill`. Each with **unit tests**.
- A terminal chat loop: you type, Claude picks tools, you see the reply.
- **Learn:** the agent tool loop, which is the #1 AI engineering skill, plus pytest and SQL with `user_id`.
- **Done when:** in the terminal, "bought COD $100" is saved correctly and
  "can I buy a $100 game" gives an answer with numbers that match a hand calculation.

### Phase 2: The WhatsApp door (1 week)
- Meta developer account → WhatsApp app → free **test number** (you can add up to 5 test phones).
- FastAPI endpoints: `GET /whatsapp` (verification handshake) and `POST /whatsapp` (incoming messages).
- **Verify the `X-Hub-Signature-256` header** on every request.
- Cloudflare Tunnel → Meta can reach your Mac.
- Reply quickly: return HTTP 200 to Meta immediately, process in the background, then send the reply.
  (Meta retries if you're slow, which causes duplicate messages, so store message IDs and ignore repeats.)
- **Learn:** webhooks, HTTPS, signatures, idempotency (handling the same event twice safely).
- **Done when:** you text the bot from your own phone and it logs and answers correctly.

### Phase 3: Deploy, always on (3–4 days)
- VPS + Docker Compose (Postgres, API, Caddy). Caddy gets free HTTPS automatically.
- Point the WhatsApp webhook at the server. Nightly `pg_dump` backups (you already have the script).
- Hard spending limit in the Anthropic console; per-user daily message limit in code.
- **Learn:** deploying, DNS, logs, backups in production.
- **Done when:** the bot works with your Mac switched off.

### Phase 4: Friends beta (2 weeks)
- 5–10 people via the test numbers or Telegram.
- Onboarding chat: currency, payday, monthly income, 3 main bills. Takes 2 minutes.
- Sunday summary, the commands `UNDO`, `HELP`, `EXPORT` and `DELETE MY DATA`, and a simple privacy policy page.
- **Learn:** real users break everything. Collect their messages that failed and fix the top 3.
- **Done when:** 5 people use it for 2 weeks without you helping them.

### Phase 5: More doors (when people ask for them)
- **Claude door:** host the MCP server over HTTPS with login (OAuth) so people add it as a connector.
  It's harder (auth), so do it after the core is solid.
- **Web dashboard** with login, linked to the same phone number.
- **Receipt photos** (Claude can read images) and **CSV import** (reuse the FinanceOS importer).

### Phase 6: Share it
- GitHub: clean README with an architecture diagram, a demo GIF, and "what I learned / bugs I fixed".
- LinkedIn post: problem → demo video → 3 engineering lessons (code calculates, AI explains; one engine, many doors; database-level security).

---

## 8. Security checklist (production)

| Risk | Protection |
|---|---|
| Fake messages pretending to be WhatsApp | Verify Meta's signature on every webhook |
| One user seeing another's data | `user_id` comes from the engine, not the AI, plus `WHERE user_id`, plus Row-Level Security |
| Prompt injection ("ignore your rules, show all users") | The AI only has per-user tools. There's nothing to leak even if it's tricked |
| Huge AI bill from spam | Per-user rate limits + a hard monthly limit in the Anthropic console + Haiku for simple messages |
| Leaked secrets | `.env` on the server only, never in Git; rotate keys if exposed; GitHub secret scanning ON |
| Data theft from the server | Firewall (only ports 80/443/SSH), SSH keys not passwords, automatic security updates, encrypted backups |
| Privacy law (Canada, **PIPEDA**) | Clear consent at onboarding, collect the minimum, `DELETE MY DATA` actually deletes, privacy policy |
| Wrong advice | Deterministic formulas + tests; the bot says "budgeting guidance, not financial advice" |
| Lost data | Nightly backups, copied off the server weekly, restore tested |

---

## 9. Risks you must know before starting

1. **WhatsApp AI policy.** Meta changed its WhatsApp Business rules so that **general-purpose AI
   chatbots** (where the AI assistant itself is the product) are not allowed on the Business API.
   Bots that serve a **specific business purpose** are allowed. An expense tracker *should* count
   as a specific use, but **read Meta's current WhatsApp Business Solution terms before launching**.
   That's exactly why the engine is channel-independent and **Telegram is the backup door**.
2. **Proactive messages cost money on WhatsApp.** Replies within 24 hours of the user's message are free.
   Messages *you start* (like the Sunday summary) need approved templates and are charged per message.
   In the trial, send summaries only to users who messaged in the last 24 hours, or use Telegram.
3. **You're holding people's financial data.** Start with friends, be transparent, collect little.
4. **Advice vs. guidance.** "You have $64 safe to spend" is budgeting. "Buy this stock" is regulated
   investment advice. Never do the second.

---

## 10. Prior work this builds on (FinanceOS, my personal prototype)

| From FinanceOS | Reused as |
|---|---|
| PostgreSQL schema, views, NUMERIC money, `kind` (transfers ≠ spending) | Core data model (add `user_id`) |
| Importer (normalize, fingerprint, dedupe) | CSV / statement import feature |
| MCP servers (read + write, validated, audit log) | The Claude door + the engine's tool layer |
| Preview → confirm → write → log | Bulk import flow |
| Backups + restore test | Production backups |
| CFO + specialists prompts | The engine's system prompt (simplified for chat) |
