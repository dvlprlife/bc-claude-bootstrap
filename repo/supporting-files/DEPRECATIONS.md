# Deprecations & Obsoletions

Tracking inventory of obsoleted elements in the **{{APP_NAME}}** AL code, so
nothing lingers past its deprecation window and removals happen on a deliberate
cadence rather than being forgotten.

## Policy

An element is retired in stages so dependent extensions and upgrade paths have
time to adapt:

1. **Pending** — `ObsoleteState = Pending` (properties) or `[Obsolete(reason, tag)]`
   (procedures). Still compiled and callable; flagged with `AL0432` at use sites.
2. **Removed** — `ObsoleteState = Removed`. The implementation is gone but the
   placeholder (and its field/object id) is retained so the id is not reused and
   upgrade code can drop the data.
3. **Deleted** — the placeholder itself is removed once no supported upgrade path
   still references it.

**Cadence:** a *Pending* element tagged at or below `(current major − 1)` is
eligible to move to *Removed*. Newer tags stay *Pending* until the next major.

**AppSourceCop constraints to know before you plan a removal:**

- A **page** cannot be set to `ObsoleteState = Removed`; obsolete it as Pending and delete it later.
- Obsoleting only a parent object does not let you delete its children (AS0031/AS0032) — obsolete each member.
- Removing an obsolete element is safe for AppSourceCop only once the **baseline** (`AppSourceCop.json`
  `version`) already shows it as obsolete.

---

## Pending — actionable removals

_Group entries by `ObsoleteTag`, newest first._

### Tag `X.Y`

| Element | File | Reason |
| --- | --- | --- |

---

## Removed placeholders

| Element | File | Removed in | Reason |
| --- | --- | --- | --- |

---

## Maintenance

- **Obsoleting** an element ⇒ add it to the matching tag cohort with its `file:line` and reason, in the same PR.
- **Removing** an element (`ObsoleteState = Removed`) ⇒ move its entry to **Removed placeholders**.
- **Deleting** a placeholder ⇒ drop the entry.
- Regenerate the raw list with a search for `ObsoleteState` and `[Obsolete(` across the AL source, then reconcile this file.
