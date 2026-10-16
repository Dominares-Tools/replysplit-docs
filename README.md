# replysplit — Split an email chain into attributed messages

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/replysplit**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

replysplit takes one incoming email and returns the conversation inside it: each message with who wrote it, when, its subject and its text, and the words the newest author typed without the quoted history. It also tells a person from a machine, recognises your own reply coming back, and pairs customer questions with your answers with personal details scrubbed. A free sift then picks the few pairs a person should review.

## Get it

Buy replysplit at https://dominares.org/tools/replysplit: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the replysplit command, with sample messages, an ingest example and the full test suite.

Requirements: The Python standard library.

This page documents version 1.1.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

Split an email reply chain into clean, attributed messages, then sift the question-and-answer pairs down to the few worth teaching a bot or a FAQ.

    replysplit tests/fixtures/outlook_reply.eml --sift --own-domain example.com

That command prints one pair marked `KEEP` (worth a person's or a model's review, which is
not the same as a lasting answer). Run it
on `tests/fixtures/gmail_reply.eml` instead and the pair is marked `DROP`, with the reason:
the reply says the order "should arrive tomorrow", which stops being true.

One incoming email usually carries the whole conversation underneath it. `replysplit`
takes it apart: each message in the chain with who wrote it, when, its subject and its
text, plus the one thing most products actually want, the words the newest author typed
(`thread.visible_text`) without the quoted history under them.

    Thanks, it arrived this morning.            <- what the newest author wrote
    On Tue, Mar 3, 2026 at 4:40 PM Help Desk <help@example.com> wrote:
    > Your order left our warehouse on Monday.  <- Help Desk, Mar 3, 4:40 PM
    >> Where is my order?                       <- Pat Doe, Mar 3, 10:15 AM

Pairs of a question and the reply it got are the raw material for a FAQ or a bot's
knowledge base, and most of them are worthless. Version 1.1 adds the sift: a free pass that
throws out what cannot teach anything, so a person reviews only the rest.

**Who it is for:** developers of help desks, shared inboxes, CRM and ticketing email
ingestion, and "answer the email with AI" products, who need to know what the customer
said this time, who said what earlier, and whether the message came from a person at all.

The package is standard library only (Python 3.10+), pure (no network, no files unless
you pass one), and never raises on malformed mail.

### What it does

- **Splits a chain into turns**, oldest first, each with sender name and address, sent
  time, subject, text, `kind` (`original`, `reply`, `forward`), `quoted` (True for every
  turn carried inside the message), and `ours` (the sender is on your side).
- **Reads the shapes real clients write.** Outlook's `From: / Sent: / To: / Subject:`
  block (one line per header, or collapsed onto one line), `-----Original Message-----`,
  `On <date>, <name> wrote:` (wrapped across lines too, and in German, French, Spanish,
  Dutch, Italian, Portuguese and Nordic forms), `---------- Forwarded message ---------`,
  `>` quote levels, HTML `<blockquote>` nesting, top-posting, bottom-posting and inline
  answers between quoted paragraphs.
- **Does not mistake prose for a header.** `From: a legal standpoint, this is fine.` and
  `On the whole, what the reporter wrote: it was fair.` stay in the text they belong to,
  and so does every line after them. A `wrote:` line counts as an attribution only when
  it carries a date, a time or an address. Nothing a person wrote is ever dropped.
- **Turns HTML into text** and keeps link text and destinations, as `text (https://...)`,
  list items as `- one` or `1. one`, and the indentation of `<pre>` blocks. Outlook Safe
  Links, Proofpoint URL Defense and Google redirects are unwrapped to the real address.
  Behind an HTML link, click trackers and personal unsubscribe links are dropped (the link
  text stays). A URL a person typed into a sentence is never touched. `display:none`
  preheaders, scripts, styles and Outlook conditional comments are dropped.
- **Removes what nobody typed:** Microsoft Graph message ids pasted into a body, "You
  don't often get email from ..." and "CAUTION: originated from outside" banners, and any
  forwarding prefix your automation adds (you name it).
- **Says whether a machine sent it** (`classify_mail`): bounces, auto-replies and
  out-of-office notes, bulk and list mail, and no-reply notifications, with the reasons in
  plain words. A customer who replies to your newsletter is still a person, and so is one
  whose mail reaches you through your own group alias (see `own_lists`).
- **Recognises the copy of your own reply** coming back in (`EchoDetector`), for a Sent
  Items sync, a BCC, or a mailbox rule, without swallowing a customer's reply that quotes
  you.
- **Gives one key per conversation** (`conversation_key`), the same through `Re:`,
  `Fwd:`, `AW:`, `SV:`, `Re[2]:` and `[EXTERNAL]`, preferring message ids when it has them.
- **Pairs questions with answers** (`question_answer_pairs`) for a FAQ, an evaluation set or
  few-shot examples, with emails, phone numbers and long digit runs scrubbed, and
  near-duplicate pairs removable.
- **Sifts pairs before a person reviews them** (`sift_pairs`, `prefilter`): drops
  out-of-office and bounce notes, acknowledgements, answers that only promise an answer,
  text that does not read as a question, answers true only for a day or week ("ships
  tomorrow"), answers that describe how your own team works, and near-duplicates of what
  you already have, each with its reason in plain words. No model is called. For the rest,
  `judge_prompt` and `parse_judgement` give you the prompt for your own model and a strict
  reader for its answer, `review_card` builds the three-part card (question, human reply,
  proposed answer) and a small JSON queue file remembers what was kept or skipped.
- **Reads broken webhook JSON** (`parse_webhook_body`): the body an automation tool posts
  when it pasted an email into a JSON template without escaping it.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

To run the tests too: `pip install ".[dev]"`.

### 60-second quickstart

    replysplit tests/fixtures/gmail_reply.eml

It prints the conversation, oldest first:

    Subject: Re: Order 1042 has not arrived
    Conversation: msg:question-0@mail.example.org
    3 turns, oldest first

    [1] original | Pat Doe <pat@example.org> | 2026-03-03 10:15 | quoted
        Hello, my order 1042 has not arrived yet. Where is it?

    [2] reply | Help Desk <help@example.com> | 2026-03-03 16:40 | quoted
        Hi Pat, your order left our warehouse on Monday and should arrive tomorrow. ...

    [3] reply | Pat Doe <pat@example.org> | 2026-03-04 09:12 | newest author's own words
        Thanks, it arrived this morning. One more question: can I change the delivery ...

Try the other flags on the same file:

    replysplit tests/fixtures/gmail_reply.eml --visible
    replysplit tests/fixtures/gmail_reply.eml --json
    replysplit tests/fixtures/outlook_reply.eml --own-domain example.com
    replysplit tests/fixtures/gmail_reply.eml --pairs --own-domain example.com
    replysplit tests/fixtures/gmail_reply.eml --sift --own-domain example.com
    replysplit tests/fixtures/gmail_reply.eml --sift --allow-time-bound --own-domain example.com
    python3 examples/sift_to_review.py tests/fixtures/outlook_reply.eml
    python3 examples/ingest_one_message.py tests/fixtures/newsletter.eml

`tests/fixtures/` holds seven sample messages: a Gmail reply, an Outlook reply, a forward,
a newsletter, a bounce, an out-of-office reply, and a message full of lines that look like
headers but are not.

### Use it in your app

```python
from replysplit import split

thread = split(open("tests/fixtures/outlook_reply.eml", "rb").read(),
               own_domains=["example.com"])

print(thread.visible_text)            # what the newest author wrote
print(thread.quoted_text)             # everything quoted, newest first
for turn in thread.turns:             # oldest first
    print(turn.kind, turn.sender, turn.sent, turn.quoted, turn.ours)
```

`split` takes the raw bytes or text of an `.eml`, an `email.message.Message`, or a body you
already extracted. Text is read as an `.eml` when its leading lines are message headers:
the usual machine-written ones (`Message-ID`, `MIME-Version`, `Received`, ...), or the
minimum you would type by hand, `From:` plus at least one of `To:`, `Subject:`, `Date:`.
Bytes are read as a message when they start with a header block holding two of those. A body that begins
with a quoted Outlook block (it has `Sent:`) stays a body. To decide for yourself, pass
`eml=True` or `eml=False`. For a bare body, say who sent it with `sender=`, `sent=` and `subject=`,
and pass the HTML part as `html=` when you have one (it is preferred over the plain text):

```python
from replysplit import split

thread = split(
    "Thanks, that fixed it.\n\nOn Mon, Mar 2, 2026 at 3:04 PM Sam <sam@example.com> wrote:\n"
    "> Try restarting the app.\n",
    sender="Pat Doe <pat@example.org>", subject="Re: App crashes")
assert thread.visible_text == "Thanks, that fixed it."
assert thread.turns[0].sender_email == "sam@example.com"
```

#### Recipes

**Is it a person?**

```python
from replysplit import classify_mail

verdict = classify_mail(open("tests/fixtures/bounce.eml", "rb").read())
print(verdict.kind)        # person, bounce, auto_reply, bulk or automated
print(verdict.reasons)     # ('it is a delivery status report', ...)
print(verdict.machine)     # True for anything but a person
```

You can also pass just what you have: `classify_mail(headers=..., sender=..., subject=...,
text=...)`. Headers are read first; text rules are narrow on purpose ("please unsubscribe me"
is a person).

**Drop the copy of a reply you sent.**

```python
from replysplit import EchoDetector, SqliteReplyStore, conversation_key

echoes = EchoDetector(SqliteReplyStore("sent.db"), own_domains=["example.com"])

# When you send a reply, record it under the conversation it belongs to.
key = conversation_key("Re: Refund", in_reply_to="<q@example.org>")
echoes.record(key, "Hi Pat, your refund was issued today.")

# When mail arrives, ask. The key defaults to the message's own conversation_key.
raw = (b"From: Help <help@example.com>\nSubject: Re: Refund\nMessage-ID: <copy@example.com>\n"
       b"In-Reply-To: <q@example.org>\n\nHi Pat, your refund was issued today.\n")
print(echoes.is_echo(raw))            # True
```

The two copies rarely share a message id, so this matches on text: the new text of the
incoming message must open with the first 120 characters of a reply you recorded, from
your side, in the same conversation, within 14 days. A customer who quotes your reply
does not match, because their own words come first, and neither does a short message
that only opens the way your reply did ("Thank you so much" against "Thank you so much for
your help with the refund, ..."): the incoming text must contain all of those first
characters. Give `own_addresses` or `own_domains` when you can; without them only the text
is compared. Replies are kept in memory
(`MemoryReplyStore`, the default), in a SQLite file (`SqliteReplyStore`), or in anything
with `add(key, text, sent_at)` and `recent(key)` methods.

**One key per conversation.**

```python
from replysplit import conversation_key

a = conversation_key("Invoice 1042")
b = conversation_key("RE: Fwd: [EXTERNAL] Re[2]: Invoice 1042")
assert a == b == "subj:invoice 1042"

# With ids it uses the root of the References chain, which is exact:
conversation_key("Re: Hi", "<r2@x.com>", "<r1@x.com>", "<root@x.com> <r1@x.com>")
# 'msg:root@x.com'
```

If your provider gives you a thread id (Gmail `threadId`, Microsoft Graph
`conversationId`), pass it as `thread_id=` and it wins.

The subject is only a fallback. Two unrelated senders who both write "Help" get the same
`subj:help`, so pass message ids or a thread id whenever you have them, and treat a
`subj:` key as a weaker match.

**Question and answer pairs for a knowledge base.**

```python
from replysplit import dedupe_pairs, question_answer_pairs

raw = open("tests/fixtures/gmail_reply.eml", "rb").read()
pairs = question_answer_pairs(raw, own_domains=["example.com"])
for p in pairs:
    print(p.question, "->", p.answer, "(handoff)" if p.handoff else "")

kept, dropped = dedupe_pairs(pairs, existing=[("Where is my order?", "It ships Monday.")])
```

A pair is one or more turns from the other side followed by your side's reply. Your own
newsletter that a customer replied to answers nothing; a bare "Thanks!" or "Sent from my
iPhone" is neither a question nor an answer; an answer that only promises one ("I'll
check and get back to you") is kept and flagged `handoff` so you can leave it out of
anything that teaches. With the default `scrub=True` the question, the answer, the subject
and both names are scrubbed (a sender known only by address shows as `[email]`);
`scrub=False` keeps personal details.

**Sift pairs before a person reviews them.**

```python
from replysplit import question_answer_pairs, sift_pairs

raw = open("tests/fixtures/gmail_reply.eml", "rb").read()
pairs = question_answer_pairs(raw, own_domains=["example.com"])
report = sift_pairs(pairs, existing=[("Where do I reset it?", "Use the Reset link.")])
print(report.counts)                      # {'kept': 0, 'time_bound': 1}
for d in report.dropped:
    print(d.code, "-", d.reason)          # time_bound - Only true for a specific day or week, ...
```

`sift_pairs` runs `prefilter` on every pair, then drops any pair whose answer restates one
already kept or already in `existing` (`dedupe_on="question"` or `"both"` compare the
question too). `existing` takes pairs, mappings with `question` and `answer`, or tuples.
Every drop has a stable `code` and a sentence for a person. The codes are `bounce`,
`auto_reply`, `no_answer`, `short_answer`, `handoff`, `no_question`, `short_question`,
`not_a_question`, `list_management`, `internal_ops`, `time_bound` and `duplicate`.

Two rules are opinions you can turn off: `time_bound=False` keeps answers that mention
"today", "tomorrow", "this week" and the like, and `internal_ops=False` keeps answers
that describe how your team works. Add words of your own with
`internal_patterns=[r"\bproject orion\b"]` (case-insensitive regular expressions).

**Ask your own model, then read its answer strictly.**

```python
import json
from replysplit import (cards_for_operator, judge_prompt, judged_card_ids, load_queue,
                        pair_id, parse_judgement, queue_counts, resolve_card, review_card,
                        upsert_cards)

pair = {"question": "How do I reset my password?",
        "answer": "Open Settings, choose Security, then Reset password. A link is emailed to you.",
        "subject": "Password help", "thread_id": "msg:abc"}

prompt = judge_prompt(pair, audience="our help-centre chatbot", reusable_topics="accounts and billing")
# Send `prompt` to your model; here is the kind of reply it should give:
reply = json.dumps({"useful": True, "reason": "Asked often.", "topic": "Reset a password",
                    "clean_answer": "Reset it under Settings, Security, Reset password."})
judgement = parse_judgement(reply)
card = review_card(pair, judgement)       # question, human reply and proposed answer, scrubbed
if card["card_id"] not in judged_card_ids("review-queue.json"):   # skip what was judged before
    upsert_cards("review-queue.json", [card])

rows = load_queue("review-queue.json")
print(queue_counts(rows))                 # {'pending': 1, 'pending_useful': 1, ...}
for c in cards_for_operator(rows):        # pending and judged useful
    print(c["original_question"], "->", c["proposed_card"])
resolve_card("review-queue.json", card["card_id"], "kept", actor="sam")
```

`judge_prompt` asks four things: will someone ask this again, is the reply a good answer,
would you be comfortable if a journalist read it, and is it still true in six months. It
scrubs personal details first and asks for JSON only. The model is yours to call;
replysplit never does.

`parse_judgement` fails closed. Text it cannot read, a "useful" verdict with no answer, the
string `"false"`, an answer that still describes how your team works, and an answer that
is only true for a day or week all come back `useful: False` with the reason, and
`clean_answer` is empty. The rules run on the model's answer too, because a prompt is an
instruction and not a guard.

**The review queue.**

The queue is one JSON file (with a small `.lock` file beside it, which you can ignore), written whole to a temporary file and moved into place, under a
lock, so a crash cannot leave half a file and two scans cannot lose each other's cards.
A conversation holds many pairs, so a card is keyed by its `card_id`, not by its
conversation: `review_card(pair, judgement, thread_id=conversation_key)` sets
`card_id = pair_id(pair, conversation_key)`, the conversation plus a fingerprint of the
question and reply, so every pair of a conversation gets its own card and the same pair
always gets the same one. `upsert_cards` never overwrites or revives a card already in the
file, and `judged_card_ids(path)` lists every pair already judged: compute `pair_id` before
you pay a model for a pair and skip it when it is in that set, so a rescan of a growing
conversation judges only what is new. (`judged_thread_ids(path)` lists conversations that
have any card; use it only when one conversation is one exchange for you.) `resolve_card`
takes the `card_id`; a `thread_id` works while exactly one card has it. A card with no
`card_id` is keyed by its `thread_id`. And `cards_for_operator(rows, include_not_useful=True)` brings back the cards the
judge set aside so a person can disagree. A card with no `useful` field is shown.
`load_queue(path, strict=True)` raises `ValueError` for a file that exists but cannot be
read, instead of showing an empty queue. `upsert_cards` and `resolve_card` read the file
that way: a queue file they cannot read raises `ValueError` and is left as it was, so
decisions are never overwritten. (`load_queue(path)` alone still reads such a file as empty,
for a review screen that must not crash.)

**Scrub personal details yourself.**

```python
from replysplit import scrub_pii

scrub_pii("Write to pat@example.org or call (619) 555-0134, account 1234 5678 9012.")
# 'Write to [email] or call [phone], account [number].'
```

**Clean HTML and links without splitting anything.**

```python
from replysplit import clean_links, html_to_text, unwrap_url

html_to_text('<p>See <a href="https://www.google.com/url?q=https%3A%2F%2Fexample.com%2Fa">the page</a></p>')
# 'See the page (https://example.com/a)'
unwrap_url("https://www.google.com/url?q=https%3A%2F%2Fexample.com%2Fa")   # 'https://example.com/a'
```

**An automation posts you mail as JSON, and sometimes it is invalid.**

```python
from replysplit import field, parse_webhook_body, text_of

payload, note = parse_webhook_body('{"subject": "Hi", "body": "He said "no" today"}')
print(payload["body"])    # He said "no" today
print(note)               # says what had to be repaired, for you to fix in the automation
subject = field(payload, "subject", "Subject")
body = text_of(field(payload, "body"))    # also unwraps {"contentType": "html", "content": "..."}
```

### Configuration reference

Everything configurable is one frozen object, `replysplit.Options`. Every function that
reads mail takes `options=` and also accepts the same fields as keyword arguments, which
win over the object. An unknown keyword raises `TypeError`. Nothing is read from the
environment unless you call `Options.from_env()`.

| Option | Default | What it does |
| --- | --- | --- |
| `own_addresses` | `()` | Addresses that are your side. A comma string or a list. |
| `own_domains` | `()` | Domains that are your side. `example.com` covers `mail.example.com`, never `notexample.com`. |
| `forward_prefixes` | `()` | Literal text your automation puts in front of a message it passes on, for example `"Newsletter feed - "`. Removed wherever it starts a line or follows a space. |
| `strip_patterns` | `()` | Regular expressions for banners your mail gateway adds (case-insensitive). Common external-sender banners are removed already. |
| `tracking_hosts` | `()` | Regular expressions matched against `host + path`, for a click-tracking service that is not built in. |
| `own_lists` | `()` | Mailing lists and group aliases that deliver mail to you, as list ids (`support.example.com`) or list addresses. Mail through one is not counted as bulk for its list headers. A list under your `own_domains`, or at one of your `own_addresses`, is recognised without being named. |
| `link_urls` | `True` | Keep an HTML link's destination after its text. `False` keeps the text only. |
| `max_turns` | `None` | Keep only the newest N turns; `thread.hidden_turns` counts the rest. |
| `max_turn_chars` | `None` | Cut a turn longer than this at a word, ending in ` …`, and set `turn.clipped`. |

Nothing is assumed to be yours until you say so: with no `own_*` setting, `turn.ours` is
always False, `question_answer_pairs` raises `ValueError`, and `--pairs` refuses.

`Options.from_env()` reads `REPLYSPLIT_OWN_ADDRESSES`, `REPLYSPLIT_OWN_DOMAINS` and
`REPLYSPLIT_OWN_LISTS` (comma lists) and `REPLYSPLIT_FORWARD_PREFIXES` (separated by `|`, because a prefix may contain a
comma). The command line reads the same variables.

#### Functions

| Function | Returns |
| --- | --- |
| `split(message, html=, sender=, sent=, subject=, eml=, options=)` | `Thread` |
| `visible_text(message, ...)` | what the newest author wrote, as a string |
| `classify_mail(message, headers=, sender=, subject=, text=, options=)` | `MailVerdict(kind, reasons, own_mailing)` with `.machine` |
| `is_machine_mail(...)` | bool |
| `EchoDetector(store, options=, window=, prefix_chars=, min_chars=)` | `.record(key, reply, sent_at)`, `.is_echo(message, key=, received_at=)` |
| `echoes_a_sent_reply(message, sent_replies)` | bool, stateless |
| `conversation_key(subject, message_id, in_reply_to, references, thread_id=)` | `str` |
| `subject_topic(subject)`, `message_ids(header)` | subject without prefixes; ids without brackets |
| `question_answer_pairs(thread, own_addresses, own_domains, scrub=True)` | `list[QAPair]` |
| `dedupe_pairs(pairs, existing=, threshold=0.5, on="both")` | `(kept, dropped)` |
| `scrub_pii(text, email=, phone=, number=)`, `similarity(a, b)` | `str`, `float` |
| `prefilter(pair, time_bound=True, internal_ops=True, internal_patterns=)` | `SiftResult(worth_judging, reason, code)` |
| `sift_pairs(pairs, existing=, time_bound=, internal_ops=, internal_patterns=, threshold=0.5, dedupe_on="answer")` | `SiftReport(kept, dropped, counts)` |
| `is_internal_ops(text, internal_patterns=)`, `is_time_bound(text)`, `narrow_question(text, newest_last=False)` | bool, bool, `str` |
| `judge_prompt(pair, audience=, reusable_topics=)`, `parse_judgement(raw, internal_patterns=)` | prompt `str`, `dict` with `useful`, `reason`, `topic`, `clean_answer` |
| `review_card(pair, judgement=None, thread_id="")`, `dedupe_cards(cards, existing=)` | `dict`, `(kept, dropped)` |
| `load_queue(path, strict=False)`, `upsert_cards`, `resolve_card`, `cards_for_operator`, `queue_counts`, `judged_thread_ids` | the JSON review queue |
| `html_to_text(html)`, `html_to_lines(html)` | text, or `(quote_depth, text)` lines |
| `unwrap_url`, `is_tracking_url`, `clean_links(text, bare=False)` | links |
| `strip_chrome`, `strip_banners`, `strip_inline_ids`, `strip_forward_prefixes` | text |
| `parse_address`, `is_own_address`, `parse_when` | `(name, address)`, bool, `datetime` or None |
| `parse_webhook_body`, `repair_json`, `field`, `text_of`, `unwrap_envelope` | see the webhook recipe |

`Thread` has `turns`, `visible_text`, `quoted_text`, `own_turn`, `latest`, `subject`,
`message_id`, `in_reply_to`, `references`, `headers`, `hidden_turns`, `conversation_key`,
`header(name)` and `to_dict()`. `Turn` has `sender_name`, `sender_email`, `sender`, `sent`
(a `datetime`, or None), `sent_utc`, `sent_text` (as written), `subject`, `to`, `text`, `kind`,
`quoted`, `ours`, `clipped` and `to_dict()`.

#### The command

    replysplit MESSAGE [--visible | --quoted | --pairs | --sift] [--json]
                       [--allow-time-bound] [--allow-internal]
                       [--html FILE] [--own-domain D] [--own-address A]
                       [--forward-prefix TEXT] [--no-link-urls] [--max-turns N]

`--visible`, `--quoted`, `--pairs` and `--sift` are alternatives; `--json` goes with any of them.
`--sift` prints each pair as `KEEP` or `DROP` with the reason (`--json` gives
`{kept, dropped, counts}`); `--allow-time-bound` and `--allow-internal` switch off those two
rules, and only apply with `--sift`. It needs `--own-domain` or `--own-address`, like `--pairs`. `MESSAGE` is an `.eml` file, a text or HTML body, or `-` for standard input. `--own-domain`,
`--own-address` and `--forward-prefix` repeat. Exit status is 0, or 2 when the input cannot
be read or the arguments are wrong.

### Honest limits

- **It reads text, not intent.** A chain with no attribution line and no quote marks (a
  person pasted an old message into the body with nothing in front of it) is one turn.
- **Attribution needs a date or an address.** `On Monday he wrote:` is left as text, and so
  is `On a related note, my manager wrote:`. A
  quoted turn whose attribution names no one has an empty sender; a name with no address
  borrows the address the same name has elsewhere in the chain, and a local part is never
  promoted to a name.
- **Dates.** English month and day names, numeric dates, RFC 5322 and Gmail, Outlook and
  Apple styles are read; anything else leaves `sent` as None and `sent_text` as written.
  Time-zone names (`PDT`) are dropped, not interpreted, so those times are naive. The
  newest turn's time comes from the `Date` header and usually has an offset, a quoted
  turn's usually does not, so comparing `turn.sent` values raw can raise `TypeError`: use
  `turn.sent_utc`, which treats a naive time as UTC (a guess, since the attribution did
  not say). A date such as `03/04/2026` with no AM/PM is ambiguous between regions and is
  not guessed; `3/4/26, at 10:15 AM` is read month first.
- **Attachments are not read.** Only the first inline `text/plain` and `text/html` parts
  are used. An attached `.eml` is not opened.
- **HTML is flattened, not rendered.** Styling is ignored except `display:none`,
  `visibility:hidden` and `mso-hide`. Quote depth comes from `<blockquote>` nesting, which is
  what Gmail, Apple Mail and Thunderbird write. Plain-text indentation is not kept; only
  the indentation of `<pre>` blocks is.
- **A subject is a weak key.** `conversation_key` falls back to the subject when there
  are no ids, and two unrelated senders can share one. A few short prefixes (`R:`, `I:`,
  `VS:`) are folded because they are reply and forward marks in some languages; `Ref:` is
  not.
- **Text rules are English.** Attribution lines are recognised in several languages, but
  the machine-mail text rules, sign-off list and `handoff` check are English. Subject
  prefixes (`Re:`, `Fwd:`, `AW:`, `WG:`, `SV:`, `VS:`, `TR:`, `RV:`, `Antw:` and a few
  more) are folded whatever the message's language.
- **Machine detection is a judgement, not a spam filter.** It names its reasons so you can
  overrule it. Text alone never makes mail bulk unless two different footer markers are
  present. Bulk mail that reaches you through a group alias is recognised as yours only
  if the `List-Id`, `List-Post` or `Mailing-list` header names a list under your
  `own_domains`, at one of your `own_addresses`, or in `own_lists`; a group that sets only
  `Precedence: list` cannot be told from a mailing list.
- **Echo detection matches on the opening text.** A reply that begins with the same 120
  characters (or with all of a shorter reply) as a canned reply you sent earlier in the same
  conversation will match it. If
  you send templated replies that open identically, record the full text and raise
  `prefix_chars`, or check the message id yourself first.
- **PII scrubbing is not a privacy-law tool.** It replaces email addresses, phone numbers
  and digit runs of seven or more. It does not find names, street addresses or anything
  spelled out in words. Review what you store.
- **Duplicate detection is word overlap** (Jaccard), good for the same question asked twice,
  not for paraphrases that share no words.
- **The sift is rules, not understanding, and English only.** It sees one pair at a time,
  from words alone. A question must start a sentence with a question word or contain a
  `?`; an answer is "time-bound" when it contains words such as today, tomorrow, yesterday,
  tonight, "this/next/last week", a weekday or a month ("May" only when capitalised), "in a
  few days" or "by the end of the week", so it will also drop a lasting answer that happens
  to say "today" (even in a greeting such as "thanks for reaching out today") (switch the rule off with `time_bound=False`). An answer is "internal"
  when it matches a short list (targeting lists, call or lead lists, talking points,
  scripts for agents, "do not tell", "off the record", "for internal use", and a few
  more); that list is a starting point, and the words your own team uses are yours to
  add with `internal_patterns`. A pair that is dropped is dropped for good unless you
  look at `report.dropped`: review it now and then.
- **A handoff is only recognised by its opening words** ("I'll check", "we will look into",
  "let me find out"), or a reply that is nothing but acknowledgement ("thanks, noted, will
  do"). A promise worded some other way is judged like any answer; so is "Sure, you can
  change it under Orders", which opens with an acknowledgement and then answers.
- **An automatic reply is recognised in the reply and the subject**, not in the question,
  so a customer who writes "I'm out of the office next month" is still asked about. A
  delivery-failure notice is recognised anywhere.
- **The model is yours, and so is its mistakes.** `parse_judgement` refuses what it can
  recognise, but "useful" from a model is still a model's opinion; a person decides
  at the end of the queue.
- **The queue file locks across processes only where `fcntl` exists** (macOS, Linux and
  other Unix). On Windows it locks across the threads of one process, and two processes
  writing one queue file can lose a write. It holds the cards, in full, so keep it where
  only your reviewers can read it: the question and reply are scrubbed of emails, phone
  numbers and digit runs, but not of names or anything spelled out in words.
- **Broken JSON repair is a heuristic.** A backslash followed by `b`, `f`, `n`, `r`, `t` or
  `u` is a valid JSON escape and is read as one, so `C:\files` arrives as a form feed. The
  returned note says when a repair happened; fix the automation when you see it.
- **Nothing caps size by default.** A 500 KB body or an 80-message digest splits in well
  under a second, but the result is as big as the input. Set `max_turns` and
  `max_turn_chars` when you ingest mail you do not control; a cut is declared
  (`hidden_turns`, `clipped`), never silent.

### Tests

    python3 -m pytest

The suite needs no network. It covers every sample under `tests/fixtures/`, a known-bad
and a known-good case for each capability, the README's own commands and code, and the
command line.

---

Docs mirror of the private repository Dominares-Tools/replysplit; updated with each release.
