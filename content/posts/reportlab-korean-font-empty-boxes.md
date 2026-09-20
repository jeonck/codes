+++
title = "Korean rendered as empty boxes in a reportlab PDF — the runtime font download was never going to work"
date = 2026-09-20
summary = "A Streamlit invoice generator downloaded NanumGothic on first use and silently fell back to Helvetica when that failed. Helvetica has no Hangul — and no won sign either, which broke English invoices too."
tags = ["reportlab", "streamlit", "fonts", "pdf", "trial-and-error"]
+++

A Streamlit app generating invoice PDFs with reportlab produced a clean-looking document in
English and an unreadable one in Korean: every Hangul character came out as a filled box.

## Symptom

Labels, company names, notes — all boxes. The numbers were fine, except each one was prefixed
by another box where `₩` should have been:

```
■■■■ ■■: INV-20260919-A01
■■■■■ (From)
■ 5,000,000
```

Nothing in the app logged an error. The English invoice rendered perfectly, so the PDF pipeline
itself was clearly working.

## Cause

The font was fetched at runtime, on first PDF generation:

```python
url = "https://fonts.google.com/download?family=Nanum+Gothic"
try:
    resp = urllib.request.urlopen(url, timeout=30)
    with zipfile.ZipFile(BytesIO(resp.read())) as zf:
        ...
except Exception:
    return False        # -> caller falls back to Helvetica
```

That URL is a browser endpoint. It does not hand a zip archive to a plain `urlopen`, and on a
host with restricted egress the connection never even completes. The download failed, the bare
`except` swallowed it, and reportlab was left with Helvetica.

Helvetica is one of the standard 14 PDF fonts, WinAnsi-encoded: no Hangul, and — the part that
is easy to miss — **no won sign** either. `₩` is U+20A9, outside WinAnsi, so an invoice in KRW
broke even with the interface in English.

Three things compounded the failure:

1. The exception was swallowed with no log and no user-visible signal.
2. The registration function set its `_FONT_REGISTERED = True` flag even on the failure path, so
   nothing ever retried and nothing could tell "registered" from "gave up".
3. The font was chosen by **interface language**, not by the text being rendered — so a Korean
   company name typed into the English interface would have broken too.

## Fix

Ship the font. A Korean TTF is ~2 MB per face, which is a reasonable thing to keep in a repo
when the alternative is a runtime dependency on a third-party endpoint:

```bash
# NanumGothic, OFL-licensed — commit the license file alongside it
curl -LO https://raw.githubusercontent.com/google/fonts/main/ofl/nanumgothic/NanumGothic-Regular.ttf
curl -LO https://raw.githubusercontent.com/google/fonts/main/ofl/nanumgothic/NanumGothic-Bold.ttf
curl -LO https://raw.githubusercontent.com/google/fonts/main/ofl/nanumgothic/OFL.txt
```

Register from the bundled path first, and keep the download only as a fallback — with a check
that what came back is actually a font, so an error page can't poison the cache:

```python
_BUNDLED_FONT_DIR = os.path.join(os.path.dirname(os.path.abspath(__file__)), "fonts")

# A TTF starts with 0x00010000 or "true"; anything else is an error page.
if data[:4] not in (b"\x00\x01\x00\x00", b"true", b"ttcf"):
    continue
```

Then choose the font by **content**, not by interface language:

```python
_HANGUL_RE = re.compile(r"[\uac00-\ud7a3\u3130-\u318f]")

needs_unicode = lang == "ko" or sym == "₩" or _has_hangul(data)
if needs_unicode and font_ok:
    fn, fn_bold = _KO_FONT_NAME, _KO_FONT_NAME_BOLD
else:
    fn, fn_bold = "Helvetica", "Helvetica-Bold"
```

And when no Korean-capable font could be registered, say so in the interface instead of handing
over a PDF full of boxes.

One more thing worth checking after the font works: Korean amounts are long. `₩12,000,000` in a
22 mm column wraps to a second line inside the table cell, which reads as a formatting bug even
though the font is now correct.

## Takeaway

A font downloaded at request time is a runtime dependency on someone else's endpoint, inside the
code path that renders your output — bundle it instead. And when a font fallback is silent, the
failure is invisible in tests (the PDF generates, the byte count looks normal) and completely
obvious to the first user who opens it. Assert on the embedded font name instead:

```python
assert b"NanumGothic" in pdf_bytes
```
