---
title: Async update writing: the BLUF format
category: productivity
tags: ['async', 'communication', 'writing']
source: Frappe community
---

Write project updates that get read by busy stakeholders using the military BLUF pattern.

## When to use
Weekly project updates, incident summaries, escalation emails, status reports — anything where the reader is busy.

## The BLUF pattern
**Bottom Line Up Front**: State the conclusion first. Details follow for those who need them.

Most people write updates chronologically (what happened, then what it means). BLUF inverts this. The most important thing is the first sentence.

## Template
```
BLUF: [One sentence — what does the reader need to know or do?]

Context (optional, 2–3 sentences): [Background for those who need it]

Details: [Specifics, metrics, timeline — only as needed]

Ask (if any): [What do you need from the reader, by when?]
```

## Example (bad — bottom-loaded)
```
Hi team, as you know we've been working on the payment integration for the past 3 weeks.
We ran into some issues with the gateway provider's sandbox environment which caused delays.
After debugging, we identified a TLS handshake issue in their test API.
They fixed it on their end yesterday. We expect to be ready for testing by Friday.
This might affect the launch timeline. Wanted to keep everyone informed.
```

## Example (good — BLUF)
```
BLUF: Payment integration is ready for QA testing by Friday — launch timeline unchanged.

Context: Gateway provider's sandbox had a TLS issue that blocked us for 2 days; they resolved it yesterday.

Ask: QA team, please schedule test run for Monday 2 Jun. Priya has the test card data.
```

## Tips
- If writing feels hard, write the BLUF sentence last, then move it to the top.
- For Slack/WhatsApp updates: bolded first line is the BLUF equivalent.
- Length rule: if the reader takes an action after reading only the first sentence, the update worked.
