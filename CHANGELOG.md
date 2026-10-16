# Changelog

## 1.1.2 (2026-10-05)

Tests only. No code change.

- Buyer test suite restored; no code change. More of the README-checking tests now ship and run in your copy. Version number only otherwise.

## 1.1.1

Fixes from a hostile review of 1.1.0.

- Several pairs from one conversation now each get a card. The queue used to key a card by
  its conversation alone, so every pair after the first was left out, and a rescan skipped
  the conversation for good. `review_card` now sets a `card_id` (new `pair_id`: the
  conversation plus a fingerprint of the question and reply); `upsert_cards`,
  `resolve_card` and the new `judged_card_ids` use it. Cards without a `card_id` are keyed
  by `thread_id` as before; `resolve_card` still accepts a `thread_id` that names one card.
  `resolve_card`'s second argument is now called `card_id`.
- `upsert_cards` and `resolve_card` no longer treat an unreadable queue file as empty and
  write over it: they raise `ValueError` and leave the file untouched. An empty file is an
  empty queue, and a scan that adds nothing no longer rewrites the file.
- A reply is a bare acknowledgement only when every word in it is one ("Thanks, noted, will
  do"). "Sure, you can change it under Orders" is an answer and is no longer dropped as a
  handoff.
- Automatic replies are recognised in the subject and the reply, not in the question, so a
  customer who says they are out of the office is not dropped.
- `sift_pairs` accounts for every input: the same pair passed twice is one kept and one
  duplicate, instead of the second vanishing from the counts.
- Documentation: the pitch no longer calls a one-off answer lasting; the package
  description mentions the sift; the example no longer claims to print the prompt and now
  skips pairs already in a queue.

## 1.1.0

The sift: decide which question-and-answer pairs are worth a person's review. Nothing in
1.0 changed; everything here is new.

- `prefilter` and `sift_pairs`: a free pass over `QAPair`s (or mappings with `question`
  and `answer`) that drops bounces and out-of-office notes, replies too short to be an
  answer, replies that only promise one, text that is not a question, list-management
  mail, answers that describe how your own team works, answers that are only true for a
  day or week, and near-duplicates of what you already have. Each drop has a stable code
  and a sentence in plain words; `SiftReport.counts` totals them.
- `is_internal_ops` and `is_time_bound`, the two rules on their own, with
  `internal_patterns=` to add your own words. `narrow_question` reads only the newest part
  of a long question.
- `judge_prompt` and `parse_judgement`: the prompt for your own model (four questions, JSON
  back) and a reader that fails closed. It also re-applies the two rules to the model's own
  answer and scrubs the stored answer. No model is called by the package.
- `review_card` (question, human reply, proposed answer, scrubbed) and `dedupe_cards`.
- The review queue: `load_queue`, `upsert_cards`, `resolve_card`, `cards_for_operator`,
  `queue_counts`, `judged_thread_ids`. One JSON file, written atomically under a lock.
- `replysplit FILE --sift` (with `--allow-time-bound` and `--allow-internal`), and
  `examples/sift_to_review.py`.
- The README opens with a pitch and a three-line example you can paste.
- The 1.0.1 entry below no longer names the publisher.

Differences from the code this was extracted from, found while porting it with tests:

- "May" counts as a month only when capitalised, so "this may take a minute" is not
  time-bound.
- "no reply" and "do not reply" in ordinary prose ("I got no reply from support") no longer
  read as an auto-reply; `no-reply@` addresses and "do not reply to this" still do.
- "I will check ..." is recognised as a handoff as well as "I'll check ...".
- `"useful": "false"` (a string) is read as false.
- "our plan" and "target customers" are not taken as internal strategy.
- A pair's question is cut from its newest end, because pairs list turns oldest first.
- The free pass also applies the time-bound and internal-operations rules to the human
  reply itself, on by default. The original applied them only to the model's cleaned answer,
  so a reply that says "today" (even in a greeting such as "Thanks for reaching out
  today!") is dropped here before any model sees it. Turn them off with `time_bound=False`
  and `internal_ops=False`, or `--allow-time-bound` and `--allow-internal`.
- The review queue is keyed by `card_id` (1.1.1), because the source had one exchange per
  thread and a conversation here has many pairs.
- Rules and examples that were specific to the original application are worded generically.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show replysplit`, `replysplit.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated (version 1.3).
- One public support contact address in the licence and the README.

## 1.0.0

First release.

- `split`: one email (raw `.eml` bytes or text, an `email.message.Message`, or a body with
  optional `html=`) into a `Thread` of ordered `Turn`s with sender name and address, sent
  time, subject, text, `kind` (original, reply, forward), `quoted` and `ours`, plus
  `visible_text` (what the newest author wrote) and `quoted_text`.
- Reads Outlook header blocks (one per line or collapsed), `-----Original Message-----`,
  `On <date>, <name> wrote:` (wrapped, and in German, French, Spanish, Dutch, Italian,
  Portuguese and Nordic forms; a `wrote:` line counts only with a date, time or address),
  forwarded-message headers, `>` quote levels, HTML
  `<blockquote>` nesting, top-posting, bottom-posting and inline answers. Prose such as
  `From: a legal standpoint` is never taken for a header.
- HTML to text that keeps link text and destinations, list items and `<pre>` indentation;
  unwraps Outlook Safe Links, Proofpoint URL Defense (v2 and v3) and Google redirects;
  drops click trackers and personal unsubscribe links behind a link, hidden preheaders and
  zero-width padding. A URL a person typed is never removed.
- Removes Microsoft Graph message ids, external-sender banners and the forwarding prefixes
  you name.
- `classify_mail`: person, bounce, auto-reply, bulk or automated, from headers first and
  the author's own words second, with reasons in plain words; flags your own mailing
  arriving back; mail through a list or group alias that delivers to you (`own_lists`, or
  a list under your domains) is not counted as bulk for its list headers.
- `Turn.sent_utc`, so turns with and without a time-zone offset can be compared.
- `split(..., eml=True|False)` overrides the guess about whether text is a whole message;
  a hand-written message with only `From:`, `To:`, `Subject:` and `Date:` is read as one.
- `EchoDetector` with `MemoryReplyStore`, `SqliteReplyStore` or your own store, and the
  stateless `echoes_a_sent_reply`.
- `conversation_key` and `subject_topic`: stable through `Re:`, `Fwd:`, `AW:`, `SV:`,
  `Re[2]:` and `[EXTERNAL]`; message ids first, provider thread id if you have one.
- `question_answer_pairs` with `scrub_pii` and `dedupe_pairs`; thank-yous and sign-offs are
  neither question nor answer, and answers that only promise one are flagged. Scrubbing
  covers the question, answer, subject and both names.
- `parse_webhook_body` and helpers: reads the JSON an automation posts when it pasted an
  email into a template unescaped, double-encoded, wrapped in a step's output, or as form
  fields, and says what it repaired.
- `Options`: one frozen configuration object, also readable from `REPLYSPLIT_*` variables.
- `replysplit` command: the turns, `--visible`, `--quoted`, `--json`, `--pairs`.
