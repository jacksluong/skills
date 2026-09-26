---
name: create-kb-md
description: Write a standalone Markdown reference note for a personal knowledge base, either distilled from the topic discussed so far in the session or built from a topic and context the user provides. Strips all one-off situational detail (the user's specific bug, project, files, people, dates) so the note reads as a general refresher months later. Use whenever the user asks to "write this up", "make a doc/note of this", "save this to my knowledge base / KB / notes / Obsidian / second brain", "turn this into a reference or cheat sheet", "document what we learned", or wants a refresher document on a topic, even if they don't say "knowledge base".
disable-model-invocation: true
---

# Create KB Markdown note

Produce one Markdown file that explains a topic well enough that the user can reread it months from now, with no memory of why they first looked into it, and get fully back up to speed.

The reader of this note is a future version of the user who has forgotten the original conversation. Every decision below follows from that.

## Step 1: Pick the mode

**Session mode.** The conversation already contains substantive discussion (a question answered, a problem debugged, a concept explained). The note distills that discussion.

**Topic mode.** The session is new, or the user names a topic and gives context (pasted text, links, notes, a short description). Build the note from their material first, then fill gaps from your own knowledge. Search the web for anything that changes over time: library versions, API signatures, prices, laws, product features. If the topic is too broad to cover in one focused note (for example "databases"), ask one question to narrow it.

If the session contains discussion but the user names a different topic, follow the named topic. If the session covered several unrelated topics, ask which one to write up, or offer one note per topic.

## Step 2: Find the general topic under the specific situation

People often learn a topic through one concrete problem. The note is about the topic, not the problem. Lift the question up one level:

| Situation in the session | Note topic |
|---|---|
| "Why does my Go worker in the billing service never exit?" | Goroutine leaks in Go: causes, detection, fixes |
| "Our Rails migration locked the orders table in prod" | Safe schema migrations on large tables |
| "Should I sell this covered call before earnings?" | Covered calls around earnings: how the mechanics and risks work |

The title names the concept. It never names the incident.

## Step 3: Strip one-off context

A sentence that depends on the original situation is useless to a reader who doesn't remember it. Remove:

- Project, company, product, repo, and service names that belong to the user
- File paths, ticket numbers, branch names, hostnames, account IDs
- Names of people involved
- Dates of the incident and any "last Tuesday" or "this sprint" timing
- Narrative of the session: "we tried X, then Y", "as discussed above", "in your case", "your code"
- Error text that only their code produces (standard error messages that anyone would see, such as `fatal error: all goroutines are asleep - deadlock!`, are fine to keep because they help recognition)

Keep the lesson. Rewrite each specific example as a minimal generic one with neutral names (`Order`, `users` table, `example.com`, `handler`). A dead end from the session stays only if it reflects a common misconception; if so, phrase it generally.

**Before:**
> In your `invoice_sync.go`, the goroutine blocked on `resultsCh` because the caller in `BillingJob` returned early on the timeout, so nobody read from the channel.

**After:**
> A goroutine that sends on an unbuffered channel blocks until someone receives. If the receiver returns early (for example on a timeout), the sender blocks forever and leaks.
> ```go
> results := make(chan int) // unbuffered
> go func() { results <- work() }() // leaks if nobody reads
> select {
> case r := <-results:
>     use(r)
> case <-time.After(time.Second):
>     return // sender is now stuck
> }
> ```
> Fix: give the channel a buffer of 1 (`make(chan int, 1)`) so the send always completes, or cancel the work with a `context.Context`.

Final check before saving: scan the draft for any word that came from the original situation and remove or generalize it.

## Step 4: Write the note

Pick the structure that best conveys this topic. The layout below is one reasonable starting point, not a required format. Reorder, merge, rename, drop, or replace sections as the content calls for. For example, a comparison of tools may read best as a table-led note, a procedure as numbered steps, and a single concept as a few short sections of prose.

```markdown
> Tags: <2-5 lowercase tags>
>
> Created: <YYYY-MM-DD>

# <Topic>

## TL;DR
- <3-6 bullets: what to remember if you read nothing else>

## Key ideas
<Each term or concept, explained in plain words.>

## How it works
<The mechanism, step by step. Add a mermaid diagram when a flow or structure is easier to see than read.>

## Examples
<Minimal, generic, runnable examples.>
```

Writing guidance:

- **Self-contained.** Define every term on first use. The reader may have forgotten the basics.
- **Knowledge only.** State facts and explanations. Never prompt the reader to act: no questions to ask anyone, next steps, things to try, or action items.
- **Final understanding only.** If the session corrected a mistake, write the corrected version. The note is a clean explanation, not a transcript.
- **Accurate.** A reference note that is wrong does damage long after the session ends. When unsure about a fact that can change, check it or say which version the note describes.
- **Code blocks** carry a language tag and stay small enough to understand at a glance.
- **Length** fits the topic. Keep it as short as the topic allows. Treat a 5 to 10 minute reread as the upper limit, and split into two notes before going past it.

## Step 5: Save and deliver

- File name: the topic in lowercase kebab-case, for example `goroutine-leaks-in-go.md`.
- Write the file. If the user has a knowledge-base folder connected or named, write it there. Otherwise deliver it with the file-delivery tool available in the session.
- In the reply, give one line on what the note covers. Don't paste the note into the chat, since the file already holds it.
