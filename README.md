# Telegram Group Management & Security Bot

A Telegram group management and security bot built with **Cloudflare Workers**, **Cloudflare D1**, and the **Telegram Bot API**. It combines rule-based moderation with optional AI-assisted analysis, and gives the group owner a full set of administrative tools inside the group and through a private chat.

The bot is designed to help group administrators maintain a safer and more organized community while reducing the need for constant manual supervision.

## 1. Project Overview

This project is a serverless Telegram bot that receives group updates through a Telegram webhook and executes moderation and administrative operations on Cloudflare Workers.

It stores configuration, moderation records, member activity, reports, scheduled messages, and custom commands in a Cloudflare D1 database, so state persists between Worker executions.

The bot serves one configured group. Its responses are written in Arabic (Palestinian dialect) and delivered through a light assistant persona. AI features are optional: the core moderation filters work without any AI provider configured.

## 2. Technologies Used

- **JavaScript** – single-file Worker implementation.
- **Cloudflare Workers** – serverless runtime for webhook and scheduled tasks.
- **Cloudflare D1** – SQL database for persistent storage.
- **Telegram Bot API** – messaging and moderation actions.
- **Cloudflare Workers AI** – optional AI inference binding.
- **Google Gemini API** – optional AI provider.
- **Grok API** – optional AI provider.

## 3. Main Features

### A. Automated Group Moderation

- **Link filtering:** removes messages containing links, including common URL formats and Telegram links.
- **Flood protection:** detects rapid message repetition (default threshold: six messages within eight seconds).
- **Forbidden-words filter:** removes messages containing words configured by the administrator.
- **Smart text filtering:** normalization and pattern checks to detect attempts to bypass the forbidden-words filter.
- **Photo rate limiting:** optional per-member limit on photos within a configurable time window; excess photos are deleted.
- **Raid detection:** alerts the owner when eight or more members join within two minutes and temporarily enforces stricter link filtering. It does not automatically ban new members.
- **Welcome messages:** customizable greeting for new members when enabled.

### B. Warnings and Member Restrictions

- Issue warnings to members and track counts in the database.
- Configurable maximum warning threshold (default: 3).
- Automatic one-hour mute when a member reaches the warning limit.
- Manual mute, unmute, ban, unban, and warning reset commands.
- Owner-exclusion checks: the moderation paths verify the target and skip the configured owner when applying warnings or restrictions.
- Moderation events are recorded for later review.

### C. Reports and Appeals

**Reports:**
- A member reports a message by replying to it with the report command.
- When an AI provider is available, the bot performs an initial review and, based on the result and a confidence threshold, may dismiss the report, delete the message, issue a warning, or refer the case for human review.
- The owner receives every report with an inline keyboard to approve or override the decision, and can revise it afterwards.

**Appeals:**
- Members can appeal a moderation action with an explanation.
- AI can provide an initial assessment when enabled.
- The final decision always remains with the owner.

### D. AI-Assisted Moderation

The project supports multiple AI providers behind one interface: **Google Gemini**, **Grok**, and **Cloudflare Workers AI**. The `TOM_AI_MODE` variable selects the provider order, with automatic fallback if a provider fails.

AI-assisted functions include:
- Classifying potentially abusive messages against configurable sensitivity thresholds.
- Detecting sustained mutual conflicts, with optional temporary auto-mute.
- Distinguishing direct personal abuse from ordinary criticism or disagreement.
- Initial review of reports and appeals.
- Understanding owner administrative requests written in natural language.

AI functionality depends on the configured credentials and per-group settings. AI classification can misjudge messages, so the design pairs it with confidence thresholds, owner review of reports and appeals, and owner approval for extensions; it is an aid to moderation, not a guarantee of correct decisions.

### E. Owner Administration

The owner can manage the group from inside the group chat or through a private chat with the bot:

- View group status, settings, warnings, and violation records.
- View activity statistics and the list of group administrators.
- Manage filters, rules, and welcome text.
- Manage custom commands and the saved knowledge base.
- Publish announcements and schedule messages.
- Review and revise reports and appeals.
- Manage all moderation settings remotely through private chat.

Sensitive operations requested in natural language are validated and require an explicit confirmation before execution.

### F. Custom Commands and Knowledge Base

**Custom commands:**
- Save text responses or Telegram media (photos, videos, documents, audio, voice) under custom command names.
- Invoke, list, and delete saved commands inside the group.

**Knowledge base:**
- Store question/answer pairs.
- Exact-match retrieval, with optional AI-assisted matching to the closest saved question.
- Update or delete stored entries.

### G. Configurable Extensions

An owner-controlled extension system can create or modify supported text and AI-based commands from a natural-language request:

1. The request is converted into a restricted, validated operation plan.
2. The plan is stored for owner review (it expires after 15 minutes).
3. The owner can approve, cancel, enable, disable, or delete the extension, or roll back the latest change.

The system only permits a predefined set of operations; it never executes arbitrary AI-generated code.

### H. Scheduled Messages and Reports

- Schedule a one-time message after a number of minutes or at a specific local time.
- Create recurring daily messages.
- List and delete scheduled tasks.
- A daily activity and moderation report is sent to the owner at 21:00 local time.

Scheduled work runs through the Worker's scheduled event handler, using the `Asia/Gaza` time zone.

## 4. Database and Data Management

Cloudflare D1 provides persistent storage for:

- Group configuration and feature settings.
- Member records and activity statistics.
- Warnings, moderation events, reports, and appeals.
- Forbidden words and custom commands.
- Saved questions and answers.
- Extension definitions, history, and pending changes.
- Scheduled and recurring messages.
- AI learning examples.

## 5. System Architecture

1. Telegram sends an update to the configured webhook.
2. The Worker validates the request secret and parses the update.
3. The bot checks permissions and group settings.
4. It performs the required moderation, database, or AI operation.
5. The result is sent back through the Telegram Bot API.

Scheduled tasks (publishing, daily reports, restriction expiry) run separately through the scheduled event handler.

## 6. Security and Privacy

- Requests to the webhook route are validated against a configured secret.
- Owner authorization checks on administrative commands.
- Restricted, validated extension operations with owner approval.
- Owner-exclusion checks in the moderation paths.
- Parameterized SQL queries through Cloudflare D1.
- API credentials are read from Worker secrets and environment bindings rather than embedded values.
- The owner's Telegram user ID and the served group's chat ID are configuration values (`OWNER_ID`, `ALLOWED_GROUP_ID`) rather than values committed in the published source.

API credentials must be stored as Cloudflare Worker secrets and never committed to a repository.

At runtime the database holds operational data such as member IDs and first names, stored message content, warnings, reports, and activity records. Anyone deploying this bot is responsible for handling that data appropriately and informing their group members.

These points describe what the source implements; they are not a substitute for a full security review.

## 7. Deployment Requirements

- A Telegram bot created through BotFather.
- A Cloudflare account with a Worker and a D1 database.
- A configured Telegram webhook pointing to the Worker's `/webhook` route.
- A Cloudflare scheduled trigger for background tasks.
- The database tables and indexes prepared according to the queries in the source.

Required configuration:

- `TELEGRAM_BOT_TOKEN` – Telegram bot token.
- `TELEGRAM_WEBHOOK_SECRET` – secret used to validate incoming webhook requests.
- `DB` – Cloudflare D1 binding.
- `OWNER_ID` – Telegram user ID of the bot owner; grants administrative access in the group and the private admin chat.
- `ALLOWED_GROUP_ID` – chat ID of the single group the bot serves; updates from other chats are ignored.

Optional AI configuration:

- `GEMINI_API_KEY` and `GEMINI_MODEL`
- `GROK_API_KEY` and `GROK_MODEL`
- `AI` – Cloudflare Workers AI binding
- `TOM_AI_MODE` – provider selection and fallback order

## 8. Project Structure

The current implementation is a single-file Cloudflare Worker containing:

- Telegram API communication and webhook handling.
- Group moderation logic.
- Owner administration (group commands and private chat).
- AI provider integration.
- Custom commands, knowledge base, and extension management.
- Database operations.
- Scheduled task handlers.

## 9. Status and Scope

- The source passes a JavaScript syntax check; a live deployment has not been independently verified as part of this publication.
- The bot is built for one group per deployment, selected through `ALLOWED_GROUP_ID`.
- AI features are optional and disabled until the corresponding providers and settings are configured.

## 10. Project Goals

- Provide a practical Telegram group security and management system.
- Automate common moderation tasks while keeping final authority with the owner.
- Give administrators useful management and scheduling tools.
- Integrate AI where it adds value, under clear limits.
- Keep persistent records through a serverless architecture.

---

**Technologies:** JavaScript · Cloudflare Workers · Cloudflare D1 · Telegram Bot API · Google Gemini · Grok · Cloudflare Workers AI

**Project Type:** Serverless Telegram Bot / Group Management / Automated Moderation
