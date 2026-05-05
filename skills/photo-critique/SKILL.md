---
name: photo-critique
description: >
  Critique a sports photography album from Google Photos. Use this skill whenever the user
  shares a Google Photos album link and wants a photo critique, shoot review, or image evaluation.
  Also trigger when the user says things like "critique my photos", "review this album",
  "evaluate my shots", "which photos are worth keeping", or any variation suggesting they
  want feedback on a set of sports photos. The skill catalogues the album via the
  google-photos-catalogue skill, selects 5–10 representative images, scores them across
  four professional categories, and appends a formatted critique session to the Photo
  Critique Journal HTML file.
---

# Photo Critique Journal — Workflow

## What this skill does

You receive a Google Photos album link (usually a lacrosse or other sports shoot). You catalogue the full album using the `google-photos-catalogue` skill to get a complete index of photos with permanent URLs, select 5–10 representative shots, evaluate each one professionally by navigating directly to it in Chrome, then append a new session block to the journal and save it so the user can open it in their browser.

## Step 1 — Gather context

Before cataloguing, confirm or ask for:
- **Album link** (required — if not already provided)
- **Sport / event name** (often inferred from the link or a previous message)
- **Date of the shoot** (infer from context or ask; format as YYYY-MM-DD)

If the user's message already contains all this, proceed directly. Don't ask for things you can figure out.

## Step 2 — Catalogue the album

Invoke the `google-photos-catalogue` skill with the album URL to get the full list of photo IDs and URLs.

```
Skill: google-photos-catalogue
Input: <album URL>
```

The skill returns a CSV with two sections. The header row:
- `album_name`, `album_date`, `album_id`, `sharing_key`, `total_photos`

Then one row per photo:
- `photo_number` — 1-based position in the album
- `photo_id` — the permanent `AF1Qip...` photo identifier
- `photo_url` — the full permanent Google Photos URL for viewing the photo

Use `total_photos` from the header to set the `badge` field in the session object (e.g. `"54 photos"`).
Use the CSV as your working index for all subsequent steps — do not browse the album grid manually.

**Note on lh3 CDN URLs:** These are not stable and must not be stored as permanent references. The `photo_url` from the catalogue is the only stable link. lh3 URLs may be extracted transiently from the page during a session for display purposes (contact sheet, evaluation screenshots) but must not appear in the scratch file's permanent data or the sessions.js entry.

## Step 3 — Scout the album

The goal is to scan the full album quickly to understand its structure and identify the 5–10 best candidates — without doing full evaluations yet.

**Build a full contact sheet in one inject.** Navigate to the album's Google Photos page. Use `javascript_tool` to extract thumbnail URLs directly from the page's embedded data script — these are transient (valid for this session only) and used purely for display. Then navigate to a neutral page (e.g. `https://www.google.com`) to avoid Google Photos' Content Security Policy and inject the contact sheet there.

**Extract lh3 URLs from the album page data script:**

```javascript
const scripts = Array.from(document.querySelectorAll('script'));
const dataScript = scripts
  .filter(s => s.textContent.includes('AF_initDataCallback'))
  .sort((a, b) => b.textContent.length - a.textContent.length)[0];

// Capture photo ID and its lh3 base URL together
const pattern = /\["(AF1Qip[A-Za-z0-9_\-]+)",\["(https:\/\/lh3\.googleusercontent\.com\/pw\/[^"=]+)/g;
const photos = [];
let m;
while ((m = pattern.exec(dataScript.textContent)) !== null) {
  photos.push({ id: m[1], lh3: m[2] });
}
window._scoutData = photos;
photos.length;
```

Match extracted photos to the catalogue CSV by `photo_id` to get the correct `photo_number` label for each cell. Use `=w200` suffix on lh3 URLs for fast loading.

**Inject the contact sheet** (after navigating to google.com):

```javascript
// _scoutData populated above; catalogue CSV loaded into memory as an array
const thumbs = catalogueRows.map(row => {
  const match = window._scoutData.find(p => p.id === row.photo_id);
  return { number: row.photo_number, url: match ? match.lh3 + '=w200' : null };
});

document.body.style.cssText = 'margin:0;background:#111;';
document.body.textContent = '';

const grid = document.createElement('div');
grid.style.cssText = 'display:grid;grid-template-columns:repeat(3,1fr);gap:4px;padding:4px;';
document.body.appendChild(grid);

thumbs.forEach(t => {
  const cell = document.createElement('div');
  cell.style.cssText = 'position:relative;aspect-ratio:4/3;overflow:hidden;background:#222;';
  const img = document.createElement('img');
  if (t.url) img.src = t.url;
  img.style.cssText = 'width:100%;height:100%;object-fit:cover;display:block;';
  const label = document.createElement('span');
  label.textContent = t.number;
  label.style.cssText = 'position:absolute;top:6px;left:6px;background:rgba(0,0,0,.75);color:#fff;font:bold 16px/1 sans-serif;padding:3px 8px;border-radius:3px;';
  cell.appendChild(img);
  cell.appendChild(label);
  grid.appendChild(cell);
});
```

Take a screenshot after injecting to confirm the grid loaded, then scroll through and take additional screenshots to cover the full album. From the contact sheet, identify which photos are worth full evaluation. Look for:
- **Coverage variety**: pre-game, peak action, celebration, sideline, post-game
- **Quality signals**: sharp vs. blurry, clean backgrounds, ball/stick in frame, readable jersey numbers
- **The album arc**: the first ~10% is often warmup; the last ~20% is often post-game or celebrations

**Save candidates to the scratch file.** Once you've scanned the full album, create `critique-scratch.json` with the session header and a `candidates` array — one entry per selected photo, nothing more:

```json
{
  "title": "EVENT NAME — CONTEXT DESCRIPTION",
  "date": "YYYY-MM-DD",
  "sport": "Lacrosse",
  "badge": "54 photos",
  "album_link": "https://photos.app.goo.gl/...",
  "candidates": [
    { "photo_number": 1,  "photo_id": "AF1Qip...", "photo_url": "https://photos.google.com/share/.../photo/AF1Qip...?key=...", "note": "warmup — #14 with ball, clean background" },
    { "photo_number": 28, "photo_id": "AF1Qip...", "photo_url": "https://photos.google.com/share/.../photo/AF1Qip...?key=...", "note": "crease scramble — multiple players, goal visible" }
  ],
  "photos": []
}
```

The `note` is a single line — just enough to remind you why you picked it. If context resets after scouting, this file tells you exactly which photos to evaluate and why, without re-scanning the album.

## Step 4 — Evaluate each candidate and save immediately

Work through `candidates` one at a time. For each:

1. Navigate to the candidate's `photo_url` in Chrome. This opens the Google Photos viewer — the full photo is displayed large and clearly, which is enough to judge all four scoring categories.
2. Once the photo loads, extract the current lh3 URL from the DOM for use as the `thumb` in the session object:
   ```javascript
   const img = document.querySelector('img[src*="lh3.googleusercontent.com/pw/"]');
   img ? img.src.replace(/=w\d+$/, '=w800') : null;
   ```
3. Take a screenshot. Evaluate the photo across all four categories (see scoring guide below).
4. **Immediately append the completed evaluation to the `photos` array** in `critique-scratch.json` and save.

Writing after each photo means a context reset loses at most one frame, not everything.

**Completed photo object format:**
```json
{
  "name": "Descriptive Title — Key Detail",
  "link": "https://photos.google.com/share/.../photo/AF1Qip...?key=...",
  "thumb": "https://lh3.googleusercontent.com/pw/...=w800",
  "verdict": "2–4 sentences of honest, specific critique.",
  "recommendation": "publish",
  "categories": {
    "technical":   { "score": 4, "notes": "..." },
    "composition": { "score": 5, "notes": "..." },
    "timing":      { "score": 4, "notes": "..." },
    "postprocess": { "score": 3, "notes": "..." }
  }
}
```

`link` is the permanent `photo_url` from the catalogue. `thumb` is the lh3 URL extracted live from the DOM during this evaluation — it's used for journal thumbnail display and is valid at time of writing.

If a photo's technical quality is ambiguous — e.g. it looks blurry but you can't tell if it's a focus miss or motion blur — click the ⓘ Info button to check the EXIF shutter speed. A blurry frame at 1/2500s is a focus miss; at 1/60s it's a shutter problem. Only do this when it would change your diagnosis.

Score each photo across four categories (1–5):

| Score | Meaning |
|-------|---------|
| 5 | Exceptional — publish immediately |
| 4 | Strong — minor fix needed |
| 3 | Adequate — noticeable issues but usable |
| 2 | Weak — significant problems |
| 1 | Reject — technical failure or wrong moment |

### Technical (sharpness, exposure, focus, motion blur)
Focus on the primary subject, shutter speed vs. motion, exposure latitude (blocked shadows, blown highlights), noise level, depth of field appropriateness.

### Composition (framing, leading lines, background, subject placement)
Subject position in frame, distracting background elements, use of natural frames (goal posts, fences, crowd), edge crops, rule of thirds, visual balance.

### Timing (decisive moment, peak action, anticipation)
Did the photographer fire at the right moment? Peak extension on a shot/save, synchronized group action, genuine emotional expression vs. before/after the moment.

### Post-process (what editing would improve it, current processing quality)
Exposure correction needed, color grading suggestions, clarity/contrast work, highlight/shadow recovery. Evaluate what it needs and whether current processing is over/under-done.

### Recommendation
- **publish** — strong enough for social media, school website, game program without qualification
- **selects** — worth keeping; good with editing or for complete coverage
- **archive** — technically fine but not needed; keep for record only

### Verdict
Write a 2–4 sentence overall assessment that a sports photographer would find genuinely useful. Be specific: name jersey numbers, describe what's in the background, explain exactly what makes the timing work or not work. Don't be vague ("good shot") — be precise ("the ball is visible mid-flight, the goalie is fully committed to the wrong side, and the shooter's follow-through is clean").

## Step 5 — Build the session object from the scratch file

Read `critique-scratch.json` to get the complete, persisted evaluation data. Convert it to a JavaScript object matching this exact schema:

```javascript
{
  title: "EVENT NAME — CONTEXT DESCRIPTION",   // e.g. "Christ School v DA — Full Game Coverage"
  date: "YYYY-MM-DD",
  sport: "Lacrosse",                            // capitalized sport name
  badge: "54 photos",                           // total_photos from catalogue header
  link: "https://photos.app.goo.gl/...",        // the original album link the user shared
  photos: [
    {
      name: "Descriptive Title — Key Detail",
      link: "https://photos.google.com/share/.../photo/AF1Qip...?key=...",  // photo_url from catalogue
      thumb: "https://lh3.googleusercontent.com/pw/...=w800",               // extracted from DOM during evaluation
      verdict: "2–4 sentences of honest, specific critique.",
      recommendation: "publish",               // or "selects" or "archive"
      categories: {
        technical:   { score: 4, notes: "Specific technical observations." },
        composition: { score: 5, notes: "Specific composition observations." },
        timing:      { score: 4, notes: "Specific timing observations." },
        postprocess: { score: 3, notes: "Specific post-processing notes." }
      }
    }
    // ... repeat for each selected photo
  ]
}
```

**Important format notes:**
- `date` must be `YYYY-MM-DD` string
- `recommendation` must be exactly one of: `"publish"`, `"selects"`, `"archive"`
- Scores are integers 1–5
- All string values use double quotes
- No trailing commas on the last item in arrays/objects (valid JSON-in-JS)

## Step 6 — Update the sessions file

The sessions data is at:
```
/Users/greg/Documents/Claude/Projects/Photo Evaluation App/sessions.js
```

Insert the new session object as the **first element** of the `window.__SESSIONS__` array (so newest appears first):

```javascript
window.__SESSIONS__ = [
  { /* NEW SESSION OBJECT */ },
  { /* existing session 1 */ },
  // ...
];
```

Use the Edit tool to make the insertion. Match on the `window.__SESSIONS__ = [` line and insert the new session object immediately after the opening bracket. After editing, read the first 20 lines back to verify the insertion looks correct.

## Step 7 — Clean up and present

Delete `critique-scratch.json` once the session object has been successfully written to `sessions.js`. Then tell the user:
- How many photos were critiqued
- The publish/selects/archive breakdown (e.g. "1 publish, 4 selects, 1 archive")
- Provide a link to open the journal:

```
[Open your Photo Critique Journal](computer:///Users/greg/Documents/Claude/Projects/Photo Evaluation App/photo-critique-journal.html)
```

The journal opens cleanly in Chrome or Safari — Google Photos thumbnails display normally when opened as a local file in the browser.

## Tips for quality critiques

- **Be a sports photographer, not a generic critic.** Think about what a working editorial sports photographer cares about: ball-in-frame, peak extension, clean background, telling the whole story of the event.
- **Jersey numbers and details matter.** "The shooter (#33)" is better than "the attacker." Name what you see.
- **Acknowledge the venue constraints.** Chain-link fences, dappled sideline light, and chaotic celebrations are all part of the sport — note when they hurt an image but don't penalize unfairly.
- **Distinguish warmup vs. game light.** Overcast warmup and directional afternoon game sun require different processing approaches.
- **Sequencing over single frames.** A large album tells a story arc; help the user understand which *types* of moments they're capturing well and where coverage has gaps.
- **Use EXIF only when it changes the diagnosis.** If a frame looks blurry and you can't tell why, check the shutter speed via the ⓘ Info button. A blurry frame at 1/2500s is a focus miss; at 1/60s it's a shutter problem. Don't open Info as a routine step.
