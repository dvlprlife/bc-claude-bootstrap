# Documentation style — Simplified Technical English (based on ASD-STE100)

**This is the writing standard for the {{APP_NAME}} user guide.** Each documentation
enhancement — a new page, a changed section, a new sentence — must follow these rules. The
standard is based on ASD-STE100 Simplified Technical English, adapted for Business Central
product documentation. This guide is the project's own adaptation; it does not claim certified
STE conformance.

Enforced through the **Documentation Maintenance**
section of `.github/copilot-instructions.md`: a PR that touches `docs/**` must keep the changed
text conformant.

## Scope

| Content | In scope? |
|---|---|
| `docs/README.md`, `docs/setup/**`, `docs/usage/**`, `docs/use-cases/**`, `docs/workflow/**` | **Yes** |
| New rows in the customer release notes (`supporting-files/release-notes/`) | **Yes** — new rows only; published rows are a historical record and stay unchanged |
| `CHANGELOG.md` entries | **Yes, where practical** — the entry is one line and the issue reference is fixed |
| `docs/appsource-listing.md` | **Yes, in part** — see **AppSource listing and app.json** below. The no-absolutes rule binds; the procedural rules do not |
| `app.json` `description` / `brief` (the published AppSource copy) | **Yes, in part** — same rule as the listing page |
| `docs/img/MANIFEST.md` | **No** — contributor capture spec, not customer documentation |
| `CLAUDE.md`, `.github/copilot-instructions.md`, `.claude/**`, `supporting-files/documentation/**` | **No** — contributor-facing |

## Writing rules

### Sentences

1. Write short sentences. Maximum **20 words** in an instruction. Maximum **25 words** in a
   description.
2. Give **one instruction for each sentence**. Two actions in sequence are two sentences.
3. Use the **active voice**. Write "The app creates one entry", not "one entry   is created".
4. Write instructions in the **imperative**: "Open the setup card." Do not write
   "You should open" or "The card can be opened".
5. Use the **present tense** for descriptions. Do not use "will" for system behavior.
6. Keep the **articles** (a, an, the). Do not delete them to make a sentence shorter.
7. Do not start an instruction with an "-ing" form. Write "Post the invoice", not
   "Posting the invoice".
8. Do not use these words and phrases: *via*, *in order to*, *leverage*, *utilize*, *commence*,
   *perform* (use *do* or the specific verb), *prior to* (use *before*), *e.g.* (use
   *for example*), *i.e.* (use *that is*), *etc.* (complete the list or write *and more* only
   when the list is open).

### Absolutes, universal qualifiers, and warranty language

**This rule outranks every other rule in this guide, including the active-voice rule.** It applies to **all** documentation the project publishes —
the release notes and the listing copy, not only this guide.

8a. **Do not make an absolute claim about how the software behaves.** Documentation is read as a
    promise. An absolute is a warranty the product must then honour in every edge case.

| Do not write | Write |
|---|---|
| it **never** deletes it | it **does not** delete it |
| the document is then **always** valid | the document then stays valid |
| **Every** entry is created when… | An entry is created when… *(or)* The app creates an entry when… |
| the copy is **fully** independent | the copy is independent |
| there is **exactly** one posting path | there is one posting path |
| the app **ensures** / **guarantees** that… | the app does… *(name the behavior)* |

8b. **Two things this rule does not touch:**
    - **Directives to the reader.** "Use a sandbox — never production" tells the reader what to do.
      It makes no claim about the software. Keep it.
    - **Factual product rules.** "A split must total 100%" is a validation rule the app
      enforces, not a promise about reliability. Keep it.

8c. **When it conflicts with rule 3 (active voice), this rule wins.** Prefer a form that satisfies
    both — "The app creates an entry when…" — but drop the absolute first.

### AppSource listing and app.json

The AppSource listing page and the `app.json`
`description` / `brief` are **customer-facing product copy**, so the standard reaches them — but
only in part, because the listing sells the product and does not instruct the reader.

8d. **Rules 8a–8c bind in full.** A claim about what {{APP_NAME}} *does* is a promise, and it is
    more exposed in the listing than in the guide: the listing is the text a buyer reads before
    they can test the claim. Write "each entry posts through a single journal gateway", not
    "**every** entry posts through…".

8e. **The procedural rules do not apply.** Rules 1–7 (sentence limits, imperative mood, one
    instruction for each sentence) and the project dictionary govern *instructions*. The listing
    has none. Marketing voice, rhetorical questions, and longer sentences are correct there.

8f. **A competitive or positioning claim is not a style decision.** "The only complete solution
    for X", "the leading Y", or a comparison to a named competitor is a commercial and legal
    claim. It is owned by the product owner, not by this guide and not by a style pass. Raise it;
    do not rewrite it.

### Paragraphs and lists

9. Keep paragraphs to a maximum of **6 sentences**. One topic for each paragraph.
10. Use a **vertical list** for more than two parallel items or steps.
11. Use a **numbered list** for a procedure. Use a bulleted list for items without an order.
12. In a warning or important note, give the **command first**, then the reason:
    "Do not use production data. The demo load changes posted entries."

### Words and terminology

13. Use **one term for one concept** through the whole guide. Do not use synonyms for variety.
14. **Technical Names are exempt** from the word rules. Quote every Business Central page,
    action, field, and report caption exactly as the application shows it, in **bold**.
    Examples: **General Ledger Setup**, **Post**, **Posting Date**.
15. Keep all numbers, amounts, expected results, status names, and code examples exactly as the
    product defines them. A style edit does not change a value.
16. Do not use a phrasal verb when one verb is available.

### Project dictionary

| Use | Do not use |
|---|---|
| open (a page or card) | navigate to, go to, launch, bring up |
| create | generate (except a **Generate** action), make, build |
| post | submit, process (for posting) |
| show / shows | display / displays, appear |
| make sure | ensure, verify (use *make sure* for a precondition; use *confirm* for a check the user does) |
| start | begin, commence, initiate |
| stop | halt, terminate |
| use | utilize, leverage |
| choose (an action) | click, press, hit |
| select (a value) | pick, tick |
| sales document | sales transaction |
| _<domain term>_ | _add the app's own domain terms here, one term for each concept_ |

> **Add the app's domain terms.** Add one row for each business term the guide uses (for
> example, the name of the app's main ledger entry). Pin the term before the guide grows, so a
> later style pass does not have to convert the whole tree.

### Structure and links

17. Do not change heading structure, anchors, file names, or image references in a style pass.
    Inbound links and `docs/img/MANIFEST.md` depend on them.
18. Every internal link must resolve. Check links after every documentation change.
19. Keep the figure placeholders and captions. A figure caption is a description — the
    25-word limit applies.

## Review checklist for a docs PR

- [ ] **No absolute or warranty claim about software behavior** (never, always, every, exactly,
      fully, ensures, guarantees) — see rules 8a–8c, and 8d–8f for the AppSource copy. Reader
      directives and validation rules are exempt.
- [ ] Changed sentences: ≤ 20 words (instructions) / ≤ 25 words (descriptions).
- [ ] One instruction for each sentence; imperative mood; active voice.
- [ ] No word from the "do not use" lists; dictionary terms used consistently.
- [ ] Technical Names match the AL captions exactly.
- [ ] No number, expected result, or link target changed by a style edit.
- [ ] Internal links resolve.
