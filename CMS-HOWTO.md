# How the CMS works (developer notes)

For Garrett — how to add a new editable field yourself.
(`KAYLA-HOW-TO-EDIT.md` is the client-facing version.)

---

## The mental model

There is **no server and no build step**. The CMS is four separate pieces that
agree on one JSON file:

```
Kayla edits at /admin
        ↓  (Decap commits the change)
GitHub: content/site.json
        ↓  (Netlify auto-deploys the repo)
Live site
        ↓  (visitor's browser fetches the JSON)
index.html JS writes the values into the page
```

So `content/site.json` is the **source of truth**, and everything else either
writes to it or reads from it.

## The four files

| File | Role |
|---|---|
| `content/site.json` | The actual data. |
| `admin/config.yml` | Describes the form Kayla sees. |
| `index.html` | An empty placeholder + JS that fills it from the JSON. |
| `admin/index.html` | The preview pane (optional, cosmetic). |

**The one rule that matters:** the `name:` in config.yml, the key in site.json,
and the key the JS reads must be **the same string**. That's the whole contract.
Most bugs are a typo across those three.

---

## Worked example: make the neighbourhood tags editable

The "Carleton Village / The Junction / Stockyards" tags are hardcoded around
line 1275 of `index.html`. Here's the full four-step change.

### 1. Add the data to `content/site.json`

```json
{
  "notice": { ... },
  "rates": { ... },
  "testimonials": [ ... ],
  "areas": ["Carleton Village", "The Junction", "Stockyards", "Toronto West End"]
}
```

### 2. Add the field to `admin/config.yml`

Add to the bottom of the `fields:` list (watch the indentation — it must line up
with the other top-level entries like `- label: "Testimonials"`):

```yaml
          - label: "Service Area Tags"
            name: "areas"
            widget: "list"
            required: false
            label_singular: "Neighbourhood"
            hint: "The little pills in the Service Area section."
```

A `list` with no `fields:` under it = a simple list of strings. (A list *with*
`fields:` gives you repeating objects, like testimonials.)

### 3. Give the HTML an id to target

```html
<div class="area-tags" id="area-tags">
  <span class="area-tag">Carleton Village</span>
  ...
</div>
```

**Leave the existing tags in place.** They are the fallback if the JSON ever
fails to load — the JS overwrites them only on success.

### 4. Fill it in from the JSON

Inside the `.then(data => { ... })` block in `index.html` (search for
`CMS CONTENT HYDRATION`, ~line 1454), add:

```js
// Service area tags
if (Array.isArray(data.areas) && data.areas.length) {
  const wrap = document.getElementById('area-tags');
  if (wrap) {
    wrap.innerHTML = data.areas
      .map(a => '<span class="area-tag">' + esc(a) + '</span>')
      .join('');
  }
}
```

Note `esc()` — it's defined at the top of that script and escapes HTML.
**Always run client-entered text through it** before using `innerHTML`,
otherwise a stray `<` in Kayla's text can break the page.

Commit, push, done. Netlify deploys in ~10s and the field shows up at /admin.

---

## Widget cheat sheet

The ones actually worth using here:

| Widget | Use for | Example in our config |
|---|---|---|
| `string` | Short single-line text | Banner text, price |
| `text` | Multi-line prose | Fine print, quote |
| `boolean` | On/off toggle | "Show the banner" |
| `list` (no `fields`) | List of plain strings | The example above |
| `list` (with `fields`) | Repeating group | Testimonials, rate line items |
| `object` | Grouping related fields | `notice`, `rates` |
| `image` | Photo upload | (none yet — lands in `intro-images/`) |
| `select` | Fixed set of choices | (none yet) |

Useful extras on any field:

- `hint:` — grey helper text under the field. Use these generously; they're the
  difference between Kayla guessing and Kayla knowing.
- `required: false` — otherwise she can't save with it empty.
- `default:` — prefilled value for new entries.

On `list` widgets specifically:

- `label_singular:` — makes the button read "Add Testimonial" not "Add Testimonials".
- `summary: "{{fields.name}} — {{fields.quote}}"` — what a collapsed row shows.
  Without this every row just says "Testimonial 1" and she has to open each one.

---

## Gotchas (things that already cost time)

1. **Netlify free tier blocks multi-contributor deploys on *private* repos.**
   CMS commits are authored by whoever is logged in, so Kayla's edits would fail
   to build. This repo is **public** for exactly that reason. Don't make it
   private again unless you upgrade to Pro.

2. **Always keep the hardcoded HTML as a fallback.** The JS only overwrites on a
   successful fetch. If you delete the static content, a bad JSON = a blank
   section for every visitor.

3. **Don't move SEO-critical text into the CMS.** Anything injected by JS isn't
   in the HTML Google reads first. Rates and testimonials are fine; the hero
   headline and main service copy should stay hardcoded.

4. **YAML indentation is load-bearing.** A misindented line in config.yml breaks
   the *entire* admin page, not just that field. Validate before pushing:
   ```bash
   npx js-yaml admin/config.yml > /dev/null && echo OK
   ```

5. **Custom preview:** `CMS.registerPreviewTemplate('content', ...)` registers by
   the *file* name (`content`), not the collection name (`site`). If you add a
   new field and want it in the preview pane, edit `admin/index.html` too —
   otherwise it just won't appear there (the live site is unaffected).

---

## Testing before you push

```bash
npx serve -l 3459 .
```

Then open http://localhost:3459 — hydration works locally because it's just a
`fetch` of a relative path. The `/admin` page will **not** work locally (it
authenticates against Netlify), so test admin changes on the deployed site.
