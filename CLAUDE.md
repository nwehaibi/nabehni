# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**نبّهني (Nabehni)** is a Telegram bot (all user-facing text and code comments are in Arabic) that tracks
product prices across e-commerce stores and alerts users by DM when the price drops. Monetization is via
paid subscription plans (free/basic/pro) sold through a Salla storefront, with **manual, admin-approved
activation** (see "Payment flow" below) — there is no payment webhook.

The entire application is one file: `nabehni_bot` (no `.py` extension, ~2500 lines). There is no package
structure, no test suite, no linter config, and no CI. Treat this file as the whole codebase.

## Commands

```bash
# Setup
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
playwright install chromium        # required for browser-based scraping fallback

# Configure secrets (never commit .env)
cp .env.example .env                # then fill in BOT_TOKEN, ADMIN_ID, SALLA_* urls

# Run the bot (long-polling, blocks)
python3 nabehni_bot
```

There are no build, lint, or test commands — none are configured in this repo. If you add tests or a
linter, add the commands here.

To sanity-check the scraping pipeline and Playwright/Chromium setup against a live bot, use the admin-only
`/diag` Telegram command (optionally `/diag <product_url>` to test a live fetch). It's the fastest way to
verify changes to the price-fetching code without writing a standalone test harness.

## Architecture

### Single-file, single-process layout

`nabehni_bot` is organized top-to-bottom as ordered sections (look for the `# ─── ... ───` / `# ═══...═══`
banner comments as section markers):

1. **Config & plans** — env vars loaded via `python-dotenv`, and `PLANS` (a dict keyed `free`/`basic`/`pro`)
   which is the single source of truth for plan limits, pricing, and marketing copy. `activate_plan()` reads
   directly from `PLANS`.
2. **SQLite persistence** (`init_db`, `get_db`, and small helper functions like `get_user`, `add_product`,
   `delete_product`, `save_order`, `save_report`) — a single `nabehni.db` file, no ORM, no migrations
   framework (schema changes are applied ad hoc via `ALTER TABLE ... EXCEPT: pass` guards in `init_db()`).
   Tables: `users`, `products`, `price_history`, `orders`, `reports`.
3. **Store detection & URL cleaning** (`detect_store_name`, `clean_tracking_url`) — a domain→Arabic-name
   lookup table, plus per-store URL-cleaning rules (Noon strips all query params, AliExpress strips
   query+fragment, Shein keeps only `main_attr`/`mallCode`, tracking params like `utm_*`/`fbclid` stripped
   generically elsewhere). Short-link domains (`a.aliexpress.com`, `amzn.to`, `sharejump`, etc.) are left
   untouched here and instead resolved by `_expand_short_url`.
4. **Price fetching cascade** (`fetch_price`, the most important function to understand before touching
   scraping logic) — tries strategies in order until one succeeds, mutating/falling back per store:
   - Noon: internal API (`fetch_price_noon_api`) tried first, free and fast.
   - AliExpress/Shein: known to block plain HTTP, so Playwright (real headless browser) is tried *first*
     (`browser_first` flag) instead of last.
   - Otherwise: `aiohttp` direct request → IPRoyal proxy → Playwright → ScraperAPI (paid, retried twice,
     tried against both expanded and original URL) → Google Search scrape (last resort).
   - Price extraction itself is store-agnostic where possible: JSON-LD (`_extract_price_from_jsonld`),
     `<meta>` tags, then store-specific fallbacks (e.g. `_extract_amazon_price`).
   - Non-SAR results are converted via `_convert_to_sar`.
   - Chromium is located dynamically across several possible install paths (`_find_chromium_executable`) —
     if Playwright-based fetching breaks, check this function and `/diag` output before assuming a code bug.
5. **Periodic price checks** (`check_prices_free` hourly, `check_prices_scraperapi` daily, registered via
   `app.job_queue.run_repeating` in `main()`). Products are split by `fetch_method` column (`free` vs
   `scraperapi`): a product automatically migrates between the two jobs based on which strategy last
   succeeded for it, so free/cheap stores aren't wastefully re-checked with paid ScraperAPI calls and vice
   versa. `_process_price_update` does the shared work of updating `current_price`/`lowest_price`, logging
   history, and deciding whether to alert (alert fires on any drop, or on hitting `target_price` if the user
   set one).
6. **Telegram handlers** — nested as closures *inside* `main()` (not top-level functions), because they
   close over `app`/`BOT_TOKEN` locals. `user_states: Dict[int, dict]` is a plain in-memory dict used as a
   hand-rolled conversation-state machine (states like `waiting_order`, `waiting_report`,
   `waiting_target_price`, `waiting_admin_reply`) instead of `python-telegram-bot`'s `ConversationHandler` —
   `handle_message` dispatches on `user_states[user_id]["state"]`. State is **not** persisted, so a bot
   restart drops any in-progress conversation.
7. `main()` wires everything: `init_db()`, builds the `Application`, registers `CommandHandler`s,
   `CallbackQueryHandler(handle_callback)` for inline-keyboard buttons, `MessageHandler` for free text, the
   two repeating jobs, and starts `run_polling()`.

### Payment flow (manual, no webhook)

Salla is used only as a storefront link, not an integrated payment API. Flow: user picks a paid plan →
bot shows the Salla product URL → user pays on Salla → user sends the Salla order number back to the bot
(`waiting_order` state) → bot forwards the order number to `ADMIN_ID` with an inline "✅ activate" button
(`adm_ok_<plan>_<user_id>` callback data) → admin taps it → `activate_plan()` runs and DMs the user. Any
change to plans, pricing, or activation must keep this manual-approval path intact unless the user
explicitly asks to add real payment-webhook integration.

### Admin surface

Commands gated on `update.effective_user.id != ADMIN_ID`: `/stats`, `/users`, `/reply <user_id> <msg>`,
`/diag`. These are the tools for operating the bot in production (checking subscriber counts, replying to
`/report` submissions, diagnosing scraping failures).

## Conventions

- All user-facing strings and in-code comments are Arabic; keep new user-facing text in Arabic and in the
  existing tone (uses emoji headers like `🔔`, `📦`, `💰` consistently in alert/status messages).
- Plan data, pricing, and product limits live only in `PLANS` — don't hardcode plan limits elsewhere.
- Store-specific scraping quirks belong in the cascade in `fetch_price` / the `_fetch_price_*` helpers, not
  scattered into handler code.
- `.env` holds all secrets (`BOT_TOKEN`, `ADMIN_ID`, `SALLA_*`, `SCRAPER_API_KEY`, `IPROYAL_PROXY*`); `.env`
  and `nabehni.db` are gitignored — never commit either.
