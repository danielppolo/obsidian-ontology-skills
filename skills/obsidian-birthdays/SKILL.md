---
name: obsidian-birthdays
description: Use when answering birthday questions from Daniel's Obsidian vault; treat People.base#Birthday as the source of truth and reproduce its filtered birthday view rather than guessing from generic searches.
---

# Obsidian Birthdays

## When To Use

Use this skill when Daniel asks about birthdays, today's birthdays, upcoming birthday reminders, or asks to evaluate the embedded base:

```markdown
![[People.base#Birthday]]
```

This skill is for reading birthday information from Daniel's Obsidian vault. Do not use it to update birthdays on contact/person notes; for that, use `obsidian-manage-person-note` and the Google Contacts-backed workflow.

## Source Of Truth

The source of truth for birthday queries is the Obsidian Base:

```text
/Users/Shared/Notes/Templates/Bases/People.base
```

The relevant embedded view is:

```markdown
![[People.base#Birthday]]
```

The `Birthday` view currently filters people notes with:

```yaml
birthday.date().day == today().day && birthday.date().month == today().month
```

It is scoped by the base-level filter:

```yaml
file.inFolder("Contacts")
```

So the intended result is: notes in `Contacts/` whose `birthday` property has the same month and day as today's date.

## Preferred Lookup Flow

1. Prefer the Obsidian CLI when available:

   ```bash
   command -v obsidian
   obsidian read path="Templates/Bases/People.base"
   ```

2. Confirm the base still contains a `Birthday` view and read its filter. Do not assume the filter if the base has changed.

3. Resolve the vault path. Daniel's usual vault is:

   ```text
   /Users/Shared/Notes
   ```

   Use `OBSIDIAN_VAULT_PATH` if set; otherwise use `/Users/Shared/Notes`.

4. Evaluate the base semantics against `Contacts/*.md`: parse frontmatter, select notes with a `birthday` date whose month/day matches today, and report name, birthday, and age when a birth year exists.

5. If the user asks for a different date, evaluate the same base semantics against that date instead of today.

## Deterministic Evaluation Snippet

Use a real date command for today; do not guess the date.

```bash
python3 - <<'PY'
import datetime, os, pathlib, re

vault = pathlib.Path(os.environ.get("OBSIDIAN_VAULT_PATH") or "/Users/Shared/Notes")
target = datetime.date.today()
root = vault / "Contacts"
rows = []

for path in root.rglob("*.md"):
    try:
        text = path.read_text(encoding="utf-8")
    except Exception:
        continue
    if not text.startswith("---"):
        continue
    end = text.find("\n---", 3)
    if end == -1:
        continue
    frontmatter = text[3:end]
    match = re.search(r'(?m)^birthday:\s*["\']?([^"\'\n#]+)', frontmatter)
    if not match:
        continue
    date_match = re.search(r'(\d{4})-(\d{1,2})-(\d{1,2})', match.group(1).strip())
    if not date_match:
        continue
    year, month, day = map(int, date_match.groups())
    if month == target.month and day == target.day:
        age = target.year - year
        rows.append((path.stem, f"{year:04d}-{month:02d}-{day:02d}", age, path.relative_to(vault).as_posix()))

for name, birthday, age, relpath in sorted(rows, key=lambda row: row[0].lower()):
    print(f"{name}\t{birthday}\t{age}\t{relpath}")
print(f"count\t{len(rows)}")
PY
```

For a requested date, replace `target = datetime.date.today()` with the explicit date, preserving the same month/day comparison.

## Reporting Style

- State that the answer comes from `People.base#Birthday`.
- Keep the output concise: name, date, age if known.
- If no birthdays match, say that the base returns no matching birthdays for the target date.
- If the base file is missing or the view changed, say so directly and report what was found instead of fabricating results.

## Common Pitfalls

1. Searching all markdown files instead of respecting the base's `Contacts/` scope.
2. Treating an embedded base as a normal note link. Read the `.base` file and evaluate the named view.
3. Guessing today's date from model context. Use `date` or Python's `datetime.date.today()` in the runtime environment.
4. Reporting birthdays from stale memory. Re-read/evaluate the base each time the user asks.
5. Confusing birthday reads with birthday edits. Edits belong to the Google Contacts/person-note workflow.

## Verification Checklist

- [ ] `Templates/Bases/People.base` was read or its relevant filter was confirmed.
- [ ] The `Birthday` view's current filter was respected.
- [ ] Only `Contacts/*.md` notes were considered unless the base changed.
- [ ] The target date came from the runtime or the user's explicit date.
- [ ] Results include enough provenance to trace back to the contact note when needed.
