# Petrol Station Shop — Automated Supplier Order Routing

An n8n workflow that automates a real small-business process: routing restock orders from shop attendants to the correct supplier, and flagging supplier replies — self-hosted end-to-end on a free cloud VPS.

## The Problem

A procurement coordinator oversees restocking across several convenience shops located at petrol stations. Shop attendants email restock requests spanning five product categories (snacks, beverages, toiletries, tobacco, bakery). The coordinator was manually reading each email, identifying the category, and emailing the relevant supplier — a repetitive process prone to delay and error, especially since a single email often requests multiple categories at once.

## The Solution

An always-on n8n workflow that:
1. Watches the coordinator's inbox for new email
2. Identifies which product categories an order mentions (an order can span multiple categories)
3. Verifies the email is genuinely from a known shop attendant (not spam, not a reply, not itself)
4. Automatically emails the correct supplier per category
5. Detects when the supplier replies and sends a flagged notification so nothing gets missed

Published and running live on the server described below — verified with unattended, end-to-end tests (a real order email sent from a separate device, with no manual interaction in n8n, correctly triggering the right supplier emails within minutes).

## Architecture

```
Email Trigger (IMAP)
   ├── If–Snacks     (body contains "Snacks"    AND sender is a known attendant) → Send Email → Supplier
   ├── If–Beverages  (body contains "Beverages" AND sender is a known attendant) → Send Email → Supplier
   ├── If–Toiletries (body contains "Toiletries"AND sender is a known attendant) → Send Email → Supplier
   ├── If–Tobacco    (body contains "Tobacco"   AND sender is a known attendant) → Send Email → Supplier
   └── If–Supplier Reply (sender is the supplier) → Send Email → Notify procurement coordinator
```

### Key design decisions

**Independent IF branches instead of a Switch node.** The initial design used a Switch node, which by default routes each item down a single matching path. Since a real order email often mentions several categories at once ("need snacks and tobacco restocked"), this meant only the first matching category would ever get emailed. Five independent IF nodes, all branching directly off the same trigger, made the multi-category logic explicit and easy to reason about — each category is checked and acted on independently, regardless of the others.

**Sender whitelist, not a blacklist.** The first version of the category checks excluded the supplier's address ("NOT from supplier"). Testing surfaced a real bug: a self-generated notification email (sent to the coordinator's own inbox) also matched the category keyword checks, since it quoted the original order text — risking a loop where the system reacted to its own notifications. Switching to a whitelist ("sender IS a known attendant") closed this and every similar edge case in one change, rather than patching exclusions one at a time.

**Self-hosted rather than a hosted automation platform**, to keep the ongoing cost at zero — running on Oracle Cloud's Always Free tier via Docker, with Caddy handling automatic HTTPS certificate issuance and renewal.

## Infrastructure

| Component | Choice | Why |
|---|---|---|
| Hosting | Oracle Cloud Always Free VPS | No time limit, no monthly cost, real dedicated server |
| Runtime | Docker Compose (n8n + Postgres + Caddy) | Isolated, reproducible, one-command redeploy |
| Domain | Free DuckDNS subdomain | No ongoing cost |
| HTTPS | Caddy, automatic Let's Encrypt certificates | Required for secure login and future OAuth integrations |

## What I'd build next

- **Payment tracking (Phase 3):** parse the amount owed from a supplier's reply and send the coordinator a clear summary, rather than automating the actual bank transfer — deliberately keeping a human in the loop for real money movement
- **WhatsApp order intake:** the original process was WhatsApp-based; adding this requires WhatsApp Business API approval (Meta Cloud API), which is a meaningfully larger scope than the email-based MVP built here
- **AI-based category detection** instead of exact keyword matching, to handle order emails that don't use the exact category words

## Notes

Supplier and attendant addresses in this repo are placeholders. Built and tested with real infrastructure and a real (anonymized) use case, not a tutorial dataset.
