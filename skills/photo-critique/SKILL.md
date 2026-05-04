---
name: photo-critique
description: >
  Critique a sports photography album from Google Photos. Use this skill whenever the user
  shares a Google Photos album link and wants a photo critique, shoot review, or image evaluation.
  Also trigger when the user says things like "critique my photos", "review this album",
  "evaluate my shots", "which photos are worth keeping", or any variation suggesting they
  want feedback on a set of sports photos. The skill browses the album in Chrome, selects
  5–10 representative images, scores them across four professional categories, and appends
  a formatted critique session to the Photo Critique Journal.
---

# Photo Critique Journal — Workflow

## What this skill does

You receive a Google Photos album link (usually a lacrosse or other sports shoot). You browse it in Chrome, select the 5–10 most representative or interesting shots, evaluate each one professionally, then append a new session block to `sessions.js` and save it so the user can open it in their browser.

## Step 1 — Gather context

Before browsing, confirm or ask for:
- **Album link** (required — if not already provided)
- **Sport / event name** (often inferred from the link or a previous message)
- **Date of the shoot** (infer from context or ask; format as YYYY-MM-DD)

If the user's message already contains all this, proceed directly. Don't ask for things you can figure out.

## Step 2 — Build the photo ID → thumb URL lookup table

**Do this first, before opening any individual photo.**

Navigate to the album grid page (`https://photos.app.goo.gl/...` redirects to `https://photos.google.com/share/...`).

### 2a — Scroll to the bottom first

Google Photos lazy-loads thumbnails. Before running any extraction, scroll all the way to the bottom of the album grid so every photo's `<img>` src is populated in the DOM. You can do this with:

```javascript
window.scrollTo(0, document.body.scrollHeight);
```

Wait a moment for images to finish loading, then scroll again if the page grew. Repeat until the page height stabilizes.

### 2b — Extract all photo IDs and thumb URLs in one shot

Run this JS from the album grid page after scrolling:

```javascript
const anchors = Array.from(document.querySelectorAll('a[href*="/photo/"]'));
const seen = new Set();
const photoMap = {};
const orderedIds = [];
for (const a of anchors) {
  const m = a.href.match(/\/photo\/([^?\/]+)/);
  if (!m) continue;
  const photoId = m[1];
  if (seen.has(photoId)) continue;
  seen.add(photoId);
  const img = a.querySelector('img[src*="lh3.googleusercontent.com"]');
  if (!img) continue;
  photoMap[photoId] = img.src.split('=')[0] + '=w800';
  orderedIds.push(photoId);
}
JSON.stringify({ count: orderedIds.length, orderedIds, photoMap });
```

This returns:
- `orderedIds` — all photo IDs in album order (use for navigation reference)
- `photoMap` — a direct `AF1Qip... → lh3...=w800` lookup table

Store both in your working memory. You'll use `photoMap` throughout the session to look up thumb URLs by photo ID without re-scraping individual photo pages.

**Notes:**
- The count may be slightly higher than the visible album size (cover image in the header links to photos too) — deduplication handles this.
- lh3 URLs are session-fresh. Collect them at the start of each critique session; they may not work if you reuse URLs from a prior session.
- Videos show a play button overlay in the grid — their entries will have no img src and are automatically excluded.

### 2c — Navigate and select photos

Click the first photo thumbnail to open it. Then use the **ArrowRight** key (one press at a time — do not batch or repeat) to advance through the album. The photo ID in the URL bar updates with each navigation.

**Photo ID and permalink structure:**
- Individual photo URL: `.../share/[ALBUM_ID]/photo/[PHOTO_ID]?key=[KEY]`
- `PHOTO_ID` starts with `AF1Qip` and is stable/permanent
- Permalink: `https://photos.google.com/share/[ALBUM_ID]/photo/[PHOTO_ID]?key=[KEY]`
- Thumb URL: look up `photoMap[PHOTO_ID]` — no additional scraping needed

**Selection criteria — aim for 5–10 in-depth photos (10 for albums of 60+ photos) that together tell the story of the shoot:**
- Pre-game (warmup, ritual, team prep)
- Peak action (the sport's defining moment — goal attempt, save, key play)
- Celebration or reaction
- Sideline / coaching moments
- Post-game (handshake line, team huddle, coach address)

Prefer variety over repetition. If there are 40 nearly identical action frames, pick the one or two where the ball, subject, and goalie/opponent are all in frame simultaneously and there's no referee obstruction.

While browsing, **also note photo IDs of up to 10 strong social media candidates** (separate from the in-depth picks). You'll use these in Step 4b.

## Step 3 — Evaluate each selected photo

For each photo, assess four categories. Score each 1–5:

| Score | Meaning |
|-------|---------|
| 5 | Exceptional — publish immediately |
| 4 | Strong — minor fix needed |
| 3 | Adequate — noticeable issues but usable |
| 2 | Weak — significant problems |
| 1 | Reject — technical failure or wrong moment |

### Technical (sharpness, exposure, focus, motion blur)
Look at: focus on the primary subject, shutter speed vs. motion, exposure latitude (blocked shadows, blown highlights), noise level, depth of field appropriateness.

### Composition (framing, leading lines, background, subject placement)
Look at: subject position in frame, distracting background elements, use of natural frames (goal posts, fences, crowd), edge crops, rule of thirds, visual balance.

### Timing (decisive moment, peak action, anticipation)
Look at: did the photographer fire at the right moment? Peak extension on a shot/save, synchronized group action, genuine emotional expression vs. before/after the moment.

### Post-process (quality of the editing and crop decisions made)
The photos you are viewing have already been cropped and processed. Evaluate the quality of those decisions — don't prescribe what to do next, assess what was done and whether it worked.

Look at: was the exposure set correctly or left too conservative/aggressive? Was contrast pushed too hard (crushed blacks, clipped highlights)? Was the white balance corrected or left with a color cast? Were distracting elements masked or selectively adjusted? Was the crop well-chosen — tight enough to eliminate waste, but not so tight it clips important elements like helmets or hands? Is the horizon level?

Write notes in evaluative language, not prescriptive:
- ✓ "The contrast was pushed too hard — blacks are crushed in the uniform shadows."
- ✓ "The crop is too tight on the right — the helmet clips the frame edge."
- ✗ "Needs more contrast." (prescriptive — avoid)

### Recommendation
- **publish** — strong enough for social media, school website, game program without qualification
- **selects** — worth keeping in the selects folder; good with editing or for complete coverage
- **archive** — technically fine but not needed; keep for record only

### Verdict
Write a 2–4 sentence overall assessment that a sports photographer would find genuinely useful. Be specific: name jersey numbers, describe what's in the background, explain exactly what makes the timing work or not work. Don't be vague ("good shot") — be precise ("the ball is visible mid-flight, the goalie is fully committed to the wrong side, and the shooter's follow-through is clean").

## Step 4 — Collect photo data

For each in-depth photo you need:
- `name`: a descriptive title you create (e.g. "Goalie Save — Full Extension Left Post")
- `link`: the full Google Photos permalink URL
- `thumb`: look up `photoMap[PHOTO_ID]` — this is the `=w800` URL you collected in Step 2b

Since you built the photoMap upfront, no additional browser scraping is needed. Just read the photo ID from the URL bar as you navigate each photo.

### Persist data to a working file immediately

Write collected photo data to disk after each photo so it survives context compression:

```python
import json, os

entry = {
  "id": 1,
  "name": "Descriptive Title — Key Detail",
  "link": "https://photos.google.com/share/.../photo/AF1Qip...",
  "thumb": "https://lh3.googleusercontent.com/pw/AP1Gcz...=w800"
}

path = os.path.join(os.getcwd(), "photo_data.json")
data = json.load(open(path)) if os.path.exists(path) else []
data.append(entry)
json.dump(data, open(path, 'w'), indent=2)
print("Saved", len(data), "photos")
```

## Step 4b — Social picks

After the in-depth set, collect up to **10 social media picks** — photos not already in the in-depth set that have immediate visual impact.

### What qualifies as a "wow" photo

A social pick must pass all three:
1. **Self-contained** — a viewer with no game context understands the emotion or action within one second
2. **Technically clean at social size** — sharp on the primary subject, exposure not blown or crushed
3. **Visually immediate** — strong colour contrast, peak-moment body language, or a striking compositional element that stops the scroll

### Extracting social pick thumbnails

Use `photoMap` from Step 2b — you already have the thumb URLs. For social picks, you can use `=w400` (smaller) since they display at reduced size in the journal:

```javascript
// On the album grid, photoMap keys are the photo IDs
// Just use: photoMap[PHOTO_ID].replace('=w800', '=w400')
```

Scroll the grid fully (Step 2a) before selecting social picks so all thumbnails are visible.

Permalink format for social picks:
```
https://photos.google.com/share/{ALBUM_ID}/photo/{PHOTO_ID}?key={KEY}
```

### Writing the social note

Each social pick gets 1–2 sentences. Focus on what makes it stop-the-scroll: "The ball is visible against the sky with the goalie fully extended — complete peak action in a single frame." Do not repeat the in-depth critique format; this is a quick editorial call, not a technical breakdown.

## Step 5 — Build the session object

Construct a JavaScript object matching this exact schema:

```javascript
{
  title: "EVENT NAME — CONTEXT DESCRIPTION",   // e.g. "Christ School v DA — Full Game Coverage"
  date: "YYYY-MM-DD",
  sport: "Lacrosse",                            // capitalized sport name
  badge: "200+ photos",                         // approximate album size, or "Game coverage" etc.
  link: "https://photos.app.goo.gl/...",        // the original album link the user shared
  photos: [
    {
      name: "Descriptive Title — Key Detail",
      link: "https://photos.google.com/share/.../photo/AF1Qip...",
      thumb: "https://lh3.googleusercontent.com/pw/...=w800",
      verdict: "2–4 sentences of honest, specific critique.",
      recommendation: "publish",               // or "selects" or "archive"
      categories: {
        technical:   { score: 4, notes: "Specific technical observations." },
        composition: { score: 5, notes: "Specific composition observations." },
        timing:      { score: 4, notes: "Specific timing observations." },
        postprocess: { score: 3, notes: "Specific post-processing notes." }
      }
    }
    // … repeat for each selected photo
  ],
  social_picks: [
    {
      name: "Brief Title",
      link: "https://photos.google.com/share/.../photo/AF1Qip...",
      thumb: "https://lh3.googleusercontent.com/pw/...=w400",
      social_note: "1–2 sentences on why this stops the scroll."
    }
    // … up to 10, none overlapping with photos[] above
  ]
}
```

**Important format notes:**
- `date` must be `YYYY-MM-DD` string
- `recommendation` must be exactly one of: `"publish"`, `"selects"`, `"archive"`
- Scores are integers 1–5
- All string values use double quotes
- No trailing commas on the last item in arrays/objects (valid JSON-in-JS)

## Step 6 — Update sessions.js

The session data lives in:
```
/Users/greg/Documents/Claude/Projects/Photo Evaluation App/sessions.js
```

The file begins with:
```javascript
window.__SESSIONS__ = [
  { /* most recent session */ },
  ...
];
```

Insert the new session object as the **first element** of the array (newest first). Use the Edit tool, matching on the `window.__SESSIONS__ = [` line and inserting immediately after the opening bracket.

After editing, verify the insertion looks correct by reading the first ~30 lines of the file back.

## Step 7 — Commit to GitHub and present

After saving sessions.js, commit and push:

```bash
cd "/sessions/adoring-optimistic-newton/mnt/Photo Evaluation App"
git add sessions.js
git commit -m "Add critique: [EVENT NAME] [DATE]"
git push
```

Then tell the user:
- How many photos were critiqued
- The publish/selects/archive breakdown (e.g. "3 publish, 4 selects")
- How many social picks were included
- Link to open the journal:

```
[Open your Photo Critique Journal](computer:///Users/greg/Documents/Claude/Projects/Photo Evaluation App/photo-critique-journal.html)
```

The journal opens cleanly in Chrome or Safari — Google Photos thumbnails display normally when opened as a local file in the browser.

## Tips for quality critiques

- **Be a sports photographer, not a generic critic.** Think about what a working editorial sports photographer cares about: ball-in-frame, peak extension, clean background, telling the whole story of the event.
- **Jersey numbers and details matter.** "The shooter (#33)" is better than "the attacker." Name what you see.
- **Acknowledge the venue constraints.** Chain-link fences, dappled sideline light, and chaotic celebrations are all part of the sport — note when they hurt an image but don't penalize unfairly.
- **Distinguish warmup vs. game light.** Overcast warmup and directional afternoon game sun call for different processing approaches — evaluate whether the edit reflected that.
- **Sequencing over single frames.** Help the user understand which *type* of moments they're capturing well and where their coverage has gaps.
- **Evaluate crops specifically.** Note when a crop is too tight (clips helmets, hands, feet), too loose (wastes frame on empty space), or would benefit from rotation to level a tilted horizon.
- **Evaluate processing decisions, not intentions.** "The contrast was pushed too hard" is a finding. Prescriptive advice ("reduce contrast") is only appropriate if the image genuinely warrants going back in Lightroom.
