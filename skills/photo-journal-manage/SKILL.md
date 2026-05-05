---
name: photo-journal-manage
description: >
  Manage the Photo Critique Journal — delete, rename, or otherwise modify existing
  sessions. Use this skill whenever the user says "delete an album", "remove a session",
  "delete from the journal", "remove a critique", or any variation asking to modify or
  remove an existing journal entry. Also trigger when the user asks to see a list of
  albums in the journal.
---

# Photo Journal Management

## Delete an album session

When the user asks to delete an album, follow these steps exactly — don't skip the confirmation.

### Step 1 — Read the session list

Read the sessions data file:
```
/Users/greg/Documents/Claude/Projects/Photo Evaluation App/sessions.js
```

Extract every session's `title`, `date`, and photo count (length of `photos` array).

### Step 2 — Present the list

Reply with a numbered list:

```
Here are the albums in your journal:

1. Christ School v DA — Full Game Coverage (2026-04-07, 10 photos)
2. Christ School v Latin — Senior Day (2026-04-25, 8 photos)
3. …

Which one would you like to delete?
```

Wait for the user to name or number their choice.

### Step 3 — Confirm

Repeat the title back and ask:

```
Delete "Christ School v DA — Full Game Coverage"? This can't be undone.
```

Wait for explicit confirmation ("yes", "delete it", "confirmed", etc.) before proceeding.

### Step 4 — Delete

Use Python via bash to remove the session from `sessions.js`. Match on the session's `title` field to locate the correct object, then remove the entire object (from its opening `{` to its closing `},`) from the `window.__SESSIONS__` array.

```python
with open('/sessions/adoring-optimistic-newton/mnt/Photo Evaluation App/sessions.js', 'r') as f:
    content = f.read()

# Find the session by title
target_title = 'EXACT TITLE HERE'
marker = f'title: "{target_title}"'
start = content.rfind('{', 0, content.find(marker))

depth = 0
i = start
while i < len(content):
    if content[i] == '{': depth += 1
    elif content[i] == '}':
        depth -= 1
        if depth == 0:
            end = i + 1
            if content[end:end+2] == ',\n': end += 2
            elif content[end] == ',': end += 1
            break
    i += 1

content = content[:start] + content[end:]

with open('/sessions/adoring-optimistic-newton/mnt/Photo Evaluation App/sessions.js', 'w') as f:
    f.write(content)
```

### Step 5 — Verify and confirm

After writing, grep `sessions.js` for the deleted title to confirm it's gone. Then tell the user:

```
Done — "Christ School v DA — Full Game Coverage" has been removed from the journal.
```

Remind them to push to GitHub if they want the live site updated.
