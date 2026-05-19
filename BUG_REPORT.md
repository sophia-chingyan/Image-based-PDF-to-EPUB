# Bug Report — PDF→EPUB Converter

## Bug 1: `docker-compose.yml` port mismatch (BREAKING)

**Severity:** High — app unreachable in local dev

The compose file maps `8000:8000`, but the Dockerfile CMD defaults to port `8080` (`${PORT:-8080}`). Since docker-compose doesn't set `PORT`, uvicorn listens on 8080 while the host-side mapping only exposes 8000→8000. Requests to `localhost:8000` hit nothing.

**Fix:** Either add `PORT=8000` to the compose environment, or change the port mapping to `8080:8080`.

---

## Bug 2: `epub_assembly.py` — each list-item wrapped in its own `<ul>` (FUNCTIONAL)

**Severity:** Medium — produces invalid/ugly EPUB HTML

Every `list-item` element emits `<ul><li>…</li></ul>` independently. Consecutive list items should share a single `<ul>` wrapper. Current output:

```html
<ul><li>Item 1</li></ul>
<ul><li>Item 2</li></ul>
```

Should be:

```html
<ul>
  <li>Item 1</li>
  <li>Item 2</li>
</ul>
```

---

## Bug 3: `structure_analysis.py` — coordinate system mismatch in `_infer_type` (FUNCTIONAL)

**Severity:** Medium — page-number and footnote heuristics silently fail

`_infer_type()` compares `tb.bbox.y0/y1` (pixel coordinates from Gemini OCR at 400 DPI, e.g. page height ~4678px) against `page_info.height` (PDF points, e.g. 842 for A4). The ratio `y_center / page_h` is therefore ~5.5× too large, so the `> 0.9` and `> 0.8` thresholds are never meaningful — `y_center` is almost always greater than `page_h`.

This means the fallback heuristics for page-number and footnote detection don't work correctly. In practice, Gemini already classifies block types, so the fallback is rarely invoked, but when it is, results are wrong.

**Fix:** Scale `page_info.height` to pixel space using `dpi / 72`, or normalise bbox coordinates.

---

## Bug 4: `pdf_assembly.py` — text-layer text placement ignores OCR bboxes (QUALITY)

**Severity:** Low-Medium — copy-paste from searchable PDF selects wrong regions

`assemble_textlayer_pdf()` distributes invisible text evenly by element index (`y_ratio = (idx + 0.5) / total`), completely ignoring the actual OCR bounding boxes. This means the invisible text doesn't align with the visible scanned text, so when a user selects text in a PDF reader, they get text from the wrong visual region.

**Fix:** Use the OCR bbox coordinates (scaled from pixel→PDF point space) to position each text overlay.

---

## Bug 5: `docker-compose.yml` — `APP_BASE_URL` not passed (FUNCTIONAL)

**Severity:** Medium — CORS and OAuth may break in local dev

The compose file passes `BASE_URL` but `Api/main.py` checks `APP_BASE_URL` first (`os.environ.get("APP_BASE_URL") or os.environ.get("BASE_URL", ...)`). The `.env.example` uses `APP_BASE_URL`. The compose environment section passes `BASE_URL=${BASE_URL}`, so if the user follows `.env.example` and sets `APP_BASE_URL`, it won't be forwarded to the container.

**Fix:** Add `APP_BASE_URL=${APP_BASE_URL}` to docker-compose environment, or normalise to one variable name.

---

## Bug 6: `SessionMiddleware` `https_only=True` blocks local dev (FUNCTIONAL)

**Severity:** Medium — login fails on localhost

`SessionMiddleware(secret_key=SECRET_KEY, https_only=True)` means the session cookie has `Secure` flag, so browsers won't send it over HTTP. Local dev at `http://localhost:8000` can't maintain sessions.

**Fix:** Make `https_only` conditional on whether `BASE_URL` starts with `https://`.

---

## Bug 7: `epub_assembly.py` — `img_inserted` set is created but never populated (COSMETIC)

**Severity:** Low — images are always re-appended at the end of each page

In `_render_page_html()`, the set `img_inserted` is initialised but never updated with `img_inserted.add(img.epub_id)`. This means the guard `if img.epub_id not in img_inserted` always passes, and images are appended even if they were already rendered inline. In practice, since the element loop never adds to `img_inserted`, all images are simply appended at the bottom — not a crash, but not the intended dedup behavior.

---

## Bug 8: `worker.py` — no protection against concurrent processing of the same job (EDGE CASE)

**Severity:** Low — if two worker instances existed (they don't currently), the same job could be processed twice

The worker doesn't atomically claim a job. It does `brpop` (atomic) then checks `status == "queued"`, but there's a TOCTOU window. With `max_concurrent_jobs: 1` and a single worker thread, this can't happen today. Noted for future multi-worker support.
