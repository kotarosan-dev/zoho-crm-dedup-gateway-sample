# Zoho CRM duplicate-lead gateway — normalization excerpt

This folder contains an **excerpt** of the normalization function from a prototype "duplicate-lead gateway" for Zoho CRM, written in Deluge.

> **Prototype, not client work.** Everything here was built and tested on 2026-11-12 in my own Zoho CRM organization (crm.zoho.jp) with synthetic test records. One Zoho Books organization was used read-only. No client data or client code is included.

## What the gateway does

Most duplicate leads are created at the door: a web form or ad integration creates the lead first and checks later, if at all. The gateway puts a function in front of Leads that:

1. normalizes the email and phone of each submission into match keys,
2. looks for existing Leads / Contacts with a strict COQL query on those keys,
3. then either updates the existing lead (only when the match is certain), creates a new lead, or sends the case to a human review queue.

Anything the gateway is not certain about goes to the queue. It never merges records automatically. A shared company phone number, for example, is treated as "possibly the same person", not "the same person".

## What is in this excerpt

`dg_normalize_excerpt.deluge` is taken from `dg_normalize` (379 lines, normalization rule v1), the single function every path of the gateway uses to build match keys. It is a pure function: no CRM or external calls.

Included:

- **Shared pre-processing**: tabs, line breaks and full-width spaces become plain spaces; full-width letters, digits and symbols become half-width, one character at a time (Deluge has no `while`, so the loop is a fixed-length list with `for each` + `break`).
- **Email**: lower-casing, rejecting `'` and `\` before any COQL use, a basic format check, an exact key, and a loose key (drops `+tag`; for Gmail also drops dots and unifies `googlemail.com`).
- **Phone / mobile**: extension split (`内線`, `ext.`, `ext`, `x`, `#`), removal of `TEL:` prefixes, hyphens, brackets and dots, Japan as the default country (`090…` → `+8190…`, `+81(0)…` handled), reason codes instead of guesses for ambiguous numbers (international prefixes, wrong length), and an 8-digit tail key.
- **Name**: concatenated last + first name with all spaces removed and lower-cased.

Omitted (marked `[省略 / OMITTED]` in the file):

- the original internal header comments,
- the half-width katakana → full-width conversion tables (used for name and company only),
- the whole company-name section (legal-form stripping such as 株式会社 / Inc.).

Because of these omissions the excerpt is reading material, not a drop-in function. Code comments are based on the original Japanese comments, lightly edited for publication; references to private design documents have been removed.

## Results from the prototype run (2026-11-12)

All figures come from one test run in the prototype org.

- Normalization self-test: **46 cases, 46 passed, 0 failed** (1.46 s). The first run after re-pasting the code was 45/1 because of one garbled full-width character during the paste; replacing that one character gave 46/46. It was not a logic defect.
- Gateway response time per submission (from the log): created 2–4 s, updated 2 s, updated + queued 3 s, queued only 1–2 s, replayed request 1.23 s.
- Representative cases behaved as designed: new lead created; empty fields filled on an exact match; a conflicting phone value queued for review while the existing value stayed unchanged; a replayed submission recorded as a duplicate with no second lead; email and phone matching two different leads → nothing created, both candidates queued; a match on Contacts → no lead created, queue item plus a task.
- Simulated rate-limit failure: retried 3 times by the retry job, then marked failed and queued; no lead created.
- Initial key backfill: about 200 records read and 166–178 updated per run, about 52–63 s per run, every run stopping on its own execution budget with 0 errors. The backfill was still in progress at the time of the report (1,937 leads processed).
- Zoho Books read-only check: 55 real customers read with GET calls only; 2 phone-match pairs found and queued for review.

Not yet verified in this prototype: calling the gateway over REST (tests ran the function from the IDE), parallel submissions, and the scheduled scan after the backfill completes.

## Article

- English: https://www.kotarosan-nocode.com/en/blog/crm-017-zoho-crm-duplicate-lead-gateway
- 日本語: _[公開後にリンクを追加]_

---

### 日本語の概要

Zoho CRM の重複リードを「作る前に止める」入口ゲートウェイの試作から、正規化関数 `dg_normalize`（379行）のうち、メール正規化・電話正規化・氏名キーと共通前処理を抜粋したものです。半角カナ変換表と会社名セクションは省略しています。筆者自身の試作組織（crm.zoho.jp、合成テストデータ）で 2026-11-12 に実装・検証したもので、受注案件のコードではありません。自己テストは46件すべて合格、ゲートウェイの応答は1件あたり1〜4秒でした。
