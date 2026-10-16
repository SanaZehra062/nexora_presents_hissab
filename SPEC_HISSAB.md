# Hisaab: a Roman Urdu Kharcha Tracker

**Tagline:** Type your spending the way you'd text a friend. Hisaab works out where your money went.

**Team name:** Nexora
**Team lead:** Sana Zehra
**Members:** Fatima Hassan, Aleeza Mujahid

---

## 1. Problem and impact

Ask a university student in Karachi where their pocket money went this month and most will shrug. They don't spend carelessly. They just never tracked it, because tracking is a chore.

Every expense app we've tried asks you to open a form, pick a category from a dropdown, type an amount and save, then repeat that after every chai and every rickshaw ride. People quit within a week. But those same people already describe their spending out loud, in a way no app understands: "aaj biryani 450, rickshaw 200, mobile load 300, kal chai samosa 120." That's Roman Urdu mixed with English, with no fixed order and no fixed spelling.

**Who it's for:** a university student or young earner in Pakistan who lives on a monthly allowance or a small salary, runs short before the month ends, and won't fill in forms to find out why.

**Why it matters:** you can't save what you can't see. If logging an expense takes five seconds and feels like sending a message, people keep doing it, and the month-end picture ("40% of my money went on food, mostly weekends") becomes something they can act on. For a student on Rs 15,000 a month, that is a real difference.

## 2. What Hisaab does

1. You type your spending in plain Roman Urdu, English or a mix, in one message or several.
2. The AI splits it into separate entries: item, amount, category and date. "Kal" becomes yesterday's date.
3. You see the parsed rows, fix anything that's off, and confirm. Nothing is saved without your OK.
4. The entries are saved to your account.
5. A dashboard shows this month's total and a breakdown by category.
6. When you want, the AI writes a short plain-language summary of your month.

## 3. The one journey that has to work

Sign up → set up 2FA → type "aaj biryani 450, rickshaw 200, mobile load 300" → see three entries → confirm → they're saved → the dashboard shows the totals → sign out → sign back in → the entries are still there, and a second account sees none of them.

We'll test this exact path first and again right before submitting.

## 4. How AI is used

Understanding messy human text is the whole product, so the AI sits in the main path every single time someone uses the app.

- **What goes in:** the raw text, today's date (so "kal" and "parso" resolve correctly), and the user's own category list.
- **What comes out:** structured JSON, a list of entries with `item`, `amount`, `category`, `date`, and a `needs_check` flag when something is ambiguous (for example "chai samosa 120" with no split, or "1.5k").
- **What we do with it:** we validate the JSON against a schema, check that every amount is a positive number, and always show the result to the user before saving. If the output is malformed we retry once, then show a clear message instead of a broken page.
- **Monthly summary:** the totals per category come from an ordinary database query. The AI only gets those numbers and writes two or three sentences about them. The arithmetic stays in code, where it's exact; the model does the part it's good at, which is language.
- **Where it runs:** server side only. The API key never reaches the browser.
- **Provider:** Anthropic or OpenAI, depending on the free credits we get at kickoff.

## 5. MVP scope (what we will build in four days)

- Sign up, sign in, TOTP 2FA, and strict per-user data separation
- Free-text entry in Roman Urdu / English
- AI parsing into entries with a review-and-edit step before saving
- Entry list with edit and delete
- Dashboard with monthly total and category breakdown chart
- AI-written monthly summary
- Default categories (Food, Transport, Mobile and data, Shopping, Bills, Other) that the user can rename

**Only if everything above is solid:** monthly budget with an overspending warning, CSV export, voice input.

**Out of scope:** bank, JazzCash or Easypaisa integration, receipt scanning, shared or family accounts, multi-currency, and any kind of financial advice. Hisaab shows where money went. It doesn't tell anyone what to invest in.

## 6. Data model

| Table | Fields |
|---|---|
| `users` | Managed by Supabase Auth (id, email, 2FA factor) |
| `categories` | id, user_id, name |
| `expenses` | id, user_id, category_id, item, amount, spent_on (date), raw_text, created_at |
| `monthly_summaries` | id, user_id, month, total, breakdown (json), ai_summary, created_at |

Row-level security is on for every table, so a user can only read or write rows where `user_id` matches their own account. Category totals are computed from `expenses` with a grouped query, so the dashboard numbers can never drift away from the entries.

## 7. Technical approach

We chose a stack that keeps the whole app in Python and hands the hard parts (login, 2FA, the database) to a managed service, so we spend our four days on the product and not on plumbing.

- **App (frontend and backend):** Python with Streamlit. The AI calls and validation run server side.
- **Database and auth:** Supabase (Postgres, Auth with TOTP 2FA, row-level security), used through its Python client
- **AI:** Anthropic or OpenAI Python SDK, with Pydantic for JSON schema validation
- **Charts:** Streamlit's built-in charts or Plotly
- **Hosting:** Streamlit Community Cloud for the app and Supabase for the database, so data survives restarts

Flow: raw text → prompt with today's date and the user's categories → model call → validate JSON → show for review → user confirms → save → dashboard query → monthly summary.

## 8. Team readiness

We are three people with complementary skills. The work splits into three areas, and each of us will take one and review the others' work, so nobody is a single point of failure. We'll confirm who owns which area at the kickoff.

- **Auth, database and deployment:** sign-up, 2FA, row-level security, the live deployment, and the two-account isolation test
- **AI parsing and backend:** the prompt, the Roman Urdu test set, JSON validation, and the monthly summary
- **Frontend and docs:** entry and review screens, dashboard and charts, README, AI disclosure, and the demo video

**Reuse plan.** We won't reinvent the boring parts. We'll use Streamlit, Plotly, Pydantic and Supabase's own auth and 2FA, and every one of them will be listed with its licence in `AI_DISCLOSURE.md`. Before writing a piece from scratch, we'll check Devmarketplace for an existing code gig that fits (auth setup, API wrapper, chart component) and credit its creator if we use it.

**Code gigs we will publish** (all written during the event and lifted out of our project):
1. Supabase Auth + TOTP 2FA + row-level security starter for Streamlit
2. Roman Urdu expense parser: the prompt, JSON schema and validator that turn messy text into structured entries
3. Category breakdown query and chart component (if time allows)

## 9. Build milestones, 19 to 22 October

1. Sign-up, sign-in, 2FA and row-level security working on a deployed app, ready to show at the kickoff checkpoint
2. AI parsing and the review-and-save step
3. Dashboard and monthly summary
4. Testing with two fresh accounts, the demo video and the docs, then submit before the deadline

We'll commit regularly across all four days and publish the code gigs as we go, not at the end.

## 10. Risks and how we'll handle them

- **Roman Urdu spelling varies a lot** ("biryani", "biriyani", "byriani"). We'll build a test set of 30 to 40 real messy inputs on day one and tune the prompt against it.
- **The model misreads an amount.** The user reviews every parsed entry before it's saved, and ambiguous ones are flagged.
- **2FA breaks late.** We build it on day one and test it every day, because it's the most common reason teams fail.
- **Scope creeps.** If it isn't in section 5, it waits until the main journey passes.

## 11. What success looks like

A judge signs up, turns on 2FA, types a messy Roman Urdu expense message, sees it split into the right entries, confirms, watches the dashboard update, signs out, signs back in, and finds everything still there. A second account sees none of it.
