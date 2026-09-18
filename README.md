# Website Vulnerabilities Demo

A Django-based web application built to demonstrate common web application vulnerabilities and their corresponding mitigations, alongside field-level database encryption.

## Overview

This project demonstrates hands-on understanding of common web application vulnerability classes — both how they're exploited and how they're properly mitigated in a real application. It consists of two demo sites, GiftcardSite and LegacySite, simulating a gift card purchasing and redemption platform.

## Vulnerabilities Demonstrated

**Cross-Site Scripting (XSS) — Reflected**
The gift card purchase and gifting pages rendered a `director` query parameter directly into the page template using Django's `|safe` filter, disabling the framework's default output escaping. This allowed arbitrary JavaScript execution via the URL, including theft of a logged-in user's session cookie.
**Fix:** removed the `safe` filter, restoring Django's default auto-escaping.

**Cross-Site Request Forgery (CSRF)**
The gifting endpoint allowed a card to be sent to any username via a simple GET request. While `SESSION_COOKIE_SAMESITE = "Lax"` blocked traditional POST-based CSRF, a malicious page could still trigger the vulnerable GET request in a victim's browser, silently gifting a card to an attacker-controlled account. A working proof-of-concept is included at "/part1/csrf/attack.html". **Fix:** added CSRF token protection and required the sender's password to authorize any gift transfer, closing the GET-based bypass.

**SQL Injection**
The "use card" feature loaded a signature field from an uploaded gift card file directly into a raw SQL query without parameterization, allowing UNION-based injection to extract salted passwords and other user data.
**Fix:** switched to Django's `.raw()` query method with proper parameter binding instead of string concatenation.

**Command Injection**
When uploaded card data failed to parse as JSON, the application fell back to a platform executable to decode it, passing the uploaded filename directly into a system call. An attacker could embed shell metacharacters in the filename to execute arbitrary commands.
**Fix:** sanitized the filename to strip non-alphanumeric characters before it reached the system call.

## Database Encryption

Implemented field-level database encryption using [`djfernet`](https://github.com/khilnani/djfernet), chosen for compatibility with the project's Django 4.1.7/Python 3 stack.

**Fields encrypted, and why:**
- **`Card.data`** — the card's usable payload, so a stolen card file can't be redeemed
- **`Card.amount`** — prevents an attacker who breaches the database from immediately identifying the highest-value cards
- **`Card.fp`** (file path) and **`Product.product_image_path`** — prevents disclosure of internal server file paths on breach
- **`Product.recommended_price`** — prevents an attacker from inferring which products are highest-value by default pricing

**What wasn't encrypted, and why:** usernames, passwords, and primary/foreign keys couldn't be encrypted due to framework constraints on indexed/relational fields — passwords are hashed instead.

**Key management:** the encryption key is loaded from an environment variable via a `.env` file (excluded from version control via `.gitignore`), with a `gen_key.py` script included so anyone cloning the repo can generate their own key. A placeholder testing key ships in `settings.py` only so the test suite runs out of the box.

## Tech Stack
- Python 3, Django 4.1.7
- SQLite
- `djfernet` — field-level database encryption
- `python-dotenv` — environment variable management for secrets

## Running Locally

```bash
git clone https://github.com/RajanBharaj/website-vulnerabilities-demo.git
cd website-vulnerabilities-demo
pip install -r requirements.txt
export DJANGO_SETTINGS_MODULE=GiftcardSite.settings   # Windows: set DJANGO_SETTINGS_MODULE=GiftcardSite.settings
python manage.py runserver
```

**Testing:** sample user accounts are seeded for local testing — see `testing.txt` for details.

## Context

The core vulnerability set (XSS, CSRF, SQL injection, command injection) was defined by an NYU Application Security coursework assignment. The database encryption implementation — including the choice of which fields to encrypt beyond the required minimum, and the environment-variable-based key management strategy — was independently designed and implemented.

## Disclaimer

This project is intentionally vulnerable and intended strictly for educational purposes. Do not deploy in a production environment.
