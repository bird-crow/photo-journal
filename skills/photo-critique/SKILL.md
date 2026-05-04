---
name: photo-critique
description: >
  Critique a sports photography album from Google Photos. Use this skill whenever the user
  shares a Google Photos album link and wants a photo critique, shoot review, or image evaluation.
  Also trigger when the user says things like "critique my photos", "review this album",
  "evaluate my shots", "which photos are worth keeping", or any variation suggesting they
  want feedback on a set of sports photos. The skill browses the album in Chrome, selects
  5–10 representative images, scores them across four professional categories, and appends
  a formatted critique session to the Photo Critique Journal HTML file.
---

# Photo Critique Journal — Workflow

## What this skill does

You receive a Google Photos album link (usually a lacrosse or other sports shoot). You browse it in Chrome, select the 5–10 most representative or interesting shots, evaluate each one professionally, then append a new session block to the journal HTML file and save it so the user can open it in their browser.

## Step 1 — Gather context

Before browsing, confirm or ask for:
- **Album link** (required — if not already provided)
- **Sport / event name** (often inferred from the link or a previous message)
- **Date of the shoot** (infer from context or ask; format as YYYY-MM-DD)

If the user's message already contains all this, proceed directly. Don't ask for things you can figure out.

## Step 2 — Browse the album in Chrome

Navigate to the album link. The typical URL pattern is `https://photos.app.goo.gl/...` which redirects to `https://photos.google.com/share/...`.

**How to navigate Google Photos:**
- The album grid shows thumbnails. Click each one to open the full photo.
- When a photo is open, the URL changes to `.../photo/AF1QipXXXXX...` — this is the individual photo's permalink.
- The thumbnail CDN URL (`lh3.googleusercontent.com/pw/...`) appears as the `src` of the `<img>` element on the photo detail page. You can read it from the page or from the DOM.
- To get the `=w800` thumbnail URL: find the img src that starts with `https://lh3.googleusercontent.com/pw/` and strip any size suffix after the last `=`, then append `=w800`.
- Videos show a play button overlay — skip them entirely.

**Selection criteria — aim for 5–10 in-depth photos (10 for albums of 60+ photos) that together tell the story of the shoot:**
- Pre-game (warmup, ritual, team prep)
- Peak action (the sport's defining moment — goal attempt, save, key play)
- Celebration or reaction
- Sideline / coaching moments
- Post-game (handshake line, team huddle, coach address)

Prefer variety over repetition. If there are 40 nearly identical action frames, pick the one or two where the ball, subject, and goalie/opponent are all in frame simultaneously and there's no referee obstruction.

While browsing the grid, **also note the photo IDs of 10 strong social media candidates** (separate from the in-depth picks). You'll collect these in Step 4b. Doing both passes together saves re-navigating the album.

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

Look at: was the exposure set correctly or left too conservative/aggressive? Was contrast pushed too hard (crushed blacks, clipped highlights)? Was the white balance corrected or left with a color cast? Were distracting elements masked or selectively adjusted? Was the crop well-chosen — tight enough to eliminate waste, but not so tight it clips important elements like helmets or hands? Is the horizon level, or does the crop need rotation?

Write notes in evaluative language, not prescriptive:
- ✓ "The contrast was pushed too hard — blacks are crushed in the uniform shadows and highlights on the white jerseys are clipping."
- ✓ "The crop is too tight on the right — the helmet clips the frame edge."
- ✓ "The shadow recovery was applied globally rather than masked to the faces, introducing noise in the skin tones."
- ✓ "The background player at right wasn't masked down; their brightness is distracting."
- ✓ "The horizon tilts a few degrees left — the crop should have been rotated to level it."
- ✗ "Needs more contrast." (prescriptive — avoid)
- ✗ "Apply shadow lift to the faces." (prescriptive — avoid)

### Recommendation
- **publish** — strong enough for social media, school website, game program without qualification
- **selects** — worth keeping in the selects folder; good with editing or for complete coverage
- **archive** — technically fine but not needed; keep for record only

### Verdict
Write a 2–4 sentence overall assessment that a sports photographer would find genuinely useful. Be specific: name jersey numbers, describe what's in the background, explain exactly what makes the timing work or not work. Don't be vague ("good shot") — be precise ("the ball is visible mid-flight, the goalie is fully committed to the wrong side, and the shooter's follow-through is clean").

## Step 4 — Capture and persist each photo's data immediately

**Do this for every photo as you visit it — do not defer URL collection to later.**

The thumbnail URL is a long opaque token that is trivially corrupted by transcription or context compression. The only safe approach is to extract it directly from the DOM and write it to disk at the moment you're on the correct page.

### Verify you're on the right photo first

Before extracting anything, confirm the current page URL contains the expected photo ID:

```javascript
// In the Chrome tab — check we're on the right photo
window.location.href.includes('AF1QipXXXXX') // replace with expected ID
```

If it returns false, wait and recheck — Google Photos sometimes takes a moment to update the URL after navigation.

### Extract the thumb URL reliably

Google Photos preloads adjacent photos in the DOM (filmstrip neighbors), so naively grabbing "any lh3 img" can return the wrong photo. Use the page's script data, which reliably contains only the current photo's URL:

```javascript
// Search script tags for the short-form lh3 base URL
const scripts = Array.from(document.querySelectorAll('script'));
let found = [];
for (const s of scripts) {
  const m = s.textContent.match(/lh3\.googleusercontent\.com\/pw\/AP1Gcz[A-Za-z0-9_\-]{30,250}/g);
  if (m) found.push(...m);
}
// Deduplicate and return
[...new Set(found)].map(u => 'https://' + u + '=w800')
```

This returns the clean base URL(s). The current photo's URL is typically the one that also appears as the largest `img` on the page. If multiple are returned, compare with `img.naturalWidth` to pick the main photo (largest).

### Write to a persistent file immediately

As soon as you have the permalink and thumb for a photo, append it to a photo data file so it survives context compression:

```python
# Run via Bash — append this photo's data to the session's working file
import json, os

entry = {
  "id": 1,
  "name": "Descriptive Title — Key Detail",
  "link": "https://photos.google.com/share/.../photo/AF1Qip...",
  "thumb": "https://lh3.googleusercontent.com/pw/AP1Gcz...=w800"
}

path = os.path.join(os.getcwd(), "photo_data.json")  # bash cwd is always the outputs directory
data = json.load(open(path)) if os.path.exists(path) else []
data.append(entry)
json.dump(data, open(path, 'w'), indent=2)
print("Saved", len(data), "photos")
```

Do this after each photo — don't wait until all photos are visited. If something goes wrong mid-session, you can resume from the file rather than re-visiting photos.

### At critique time, read the file

When writing critiques (Step 3) and building the session object (Step 5), read `photo_data.json` to get the `link` and `thumb` values rather than reconstructing from memory. This eliminates transcription errors entirely.

## Step 4b — Social picks: grid thumbnails and wow-factor notes

After completing the in-depth photo data file, do a second pass to collect up to **10 social media picks** — photos not already in the in-depth set that have immediate visual impact.

### What qualifies as a "wow" photo

A social pick must pass all three of these tests:
1. **Self-contained** — a viewer with no game context understands the emotion or action within one second. A ball mid-flight, a goalie fully extended, a player's raw expression — these need no caption.
2. **Technically clean at social size** — sharp on the primary subject, exposure not blown or crushed, nothing catastrophically cropped (heads, feet intact). Minor background distractions are acceptable; focus misses are not.
3. **Visually immediate** — strong colour contrast, peak-moment body language, or a striking compositional element (symmetry, leading lines, frame-within-frame) that makes it stop the scroll.

Photos already selected for in-depth review should not appear here — assume all in-depth picks will be shared.

### Extracting grid thumbnails

Grid thumbnails are fast to collect without navigating to each photo page. From the album grid, run:

```javascript
// Extract photo IDs and grid thumbnail URLs in one pass
const anchors = Array.from(document.querySelectorAll('a[href*="/photo/"]'));
anchors.map(a => {
  const id = a.href.match(/\/photo\/(AF1Qip[^?]+)/)?.[1];
  const img = a.querySelector('img');
  const src = img?.getAttribute('src') || '';
  // Strip size suffix and replace with =w400
  const base = src.match(/^(https:\/\/lh3\.googleusercontent\.com\/pw\/[^=]+)/)?.[1];
  return { id, thumb: base ? base + '=w400' : null };
}).filter(x => x.id && x.thumb)
```

Scroll the grid first to load all photos, then run this. The result gives you all visible photo IDs and their grid-quality thumbnails (`=w400`). Pick your 10 candidates from this list by ID, note their grid thumbnails, and construct the permalink for each:
```
https://photos.google.com/share/{ALBUM_ID}/photo/{PHOTO_ID}?key={KEY}
```

Grid thumbnails at `=w400` are lower resolution than the `=w800` used for in-depth photos — this is intentional since they display smaller in the social picks table. If a grid thumb fails to load in the journal, fall back to navigating to the photo page and using the script-tag extraction method from Step 4.

### Persist social picks to the data file

Append social picks to `photo_data.json` with a `type` field to distinguish them:

```python
import json, os

picks = [
  { "type": "social", "id": 1, "name": "Brief title", "link": "https://photos.google.com/...", "thumb": "https://lh3...=w400", "social_note": "One or two sentences on the wow factor." },
  # ... up to 10
]

path = os.path.join(os.getcwd(), "photo_data.json")
data = json.load(open(path)) if os.path.exists(path) else []
data.extend(picks)
json.dump(data, open(path, 'w'), indent=2)
```

### Writing the social note

Each social pick gets 1–2 sentences. Focus on what makes it stop-the-scroll: "The ball is visible against the sky with the goalie fully extended — complete peak action in a single frame." or "Three players' expressions in the same shot — the scorer's elation, the defender's dejection, the keeper's disbelief — tell the whole story." Do not repeat the in-depth critique format; this is a quick editorial call, not a technical breakdown.

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
      thumb: "https://lh3.googleusercontent.com/pw/...=w400",   // grid thumbnail, smaller than in-depth
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

## Step 6 — Update the HTML journal file

The journal is at:
```
/Users/greg/Documents/Claude/Projects/Photo Evaluation App/photo-critique-journal.html
```

Find this comment in the file:
```html
<!-- SESSIONS DATA — Claude updates this block when adding new critiques -->
<script>
window.__SESSIONS__ = [
```

Insert the new session object as the **first element** of the array (so newest appears first). The structure is:
```javascript
window.__SESSIONS__ = [
  { /* NEW SESSION OBJECT */ },
  { /* existing session 1 */ },
  // ...
];
```

Use the Edit tool to make the insertion. Match on the `window.__SESSIONS__ = [` line and insert the new session object immediately after the opening bracket.

After editing, verify the insertion looks correct by reading the relevant section back.

## Step 7 — Confirm and present

After saving, tell the user:
- How many photos were critiqued
- The publish/selects/archive breakdown (e.g. "3 publish, 4 selects")
- Provide a link to open the journal:

```
[Open your Photo Critique Journal](computer:///Users/greg/Documents/Claude/Projects/Photo Evaluation App/photo-critique-journal.html)
```

The journal opens cleanly in Chrome or Safari — Google Photos thumbnails display normally when opened as a local file in the browser.

## Tips for quality critiques

- **Be a sports photographer, not a generic critic.** Think about what a working editorial sports photographer cares about: ball-in-frame, peak extension, clean background, telling the whole story of the event.
- **Jersey numbers and details matter.** "The shooter (#33)" is better than "the attacker." Name what you see.
- **Acknowledge the venue constraints.** Chain-link fences, dappled sideline light, and chaotic celebrations are all part of the sport — note when they hurt an image but don't penalize unfairly.
- **Distinguish warmup vs. game light.** Overcast warmup and directional afternoon game sun call for different processing approaches — evaluate whether the edit reflected that.
- **Sequencing over single frames.** The user shot 200+ images; help them understand which *type* of moments they're capturing well and where their coverage has gaps.
- **Evaluate crops specifically.** Note when a crop is too tight (clips helmets, hands, feet), too loose (wastes frame on empty space), or would benefit from rotation to level a tilted horizon.
- **Evaluate processing decisions, not intentions.** You are seeing the finished file. "The contrast was pushed too hard" is a finding. "Consider reducing contrast" is advice for a re-edit — use it only if the image genuinely warrants going back in.
