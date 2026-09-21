+++
title = "The sign-in gate is the one screen every visitor sees — mine said only \"sign in\""
date = 2026-09-21
summary = "A gated app shows its login screen to everyone and its actual product to almost nobody. Putting a live sample on that screen took a few lines — and immediately exposed a CDN that had been failing silently for anyone whose network was not mine."
tags = ["streamlit", "ux", "oauth", "frontend", "trial-and-error"]
+++

An invoice tool sits behind Google sign-in. Which means every person who arrives sees the gate,
and only the ones who already decided to trust it see the product. The gate was the most-viewed
screen in the app and the least designed one — a title, a caption, a button.

## What a bare gate costs

```python
if not is_signed_in():
    if st.button("Sign in with Google", type="primary"):
        st.login("google")
    return False
```

Two things go wrong there, and neither shows up as an error.

A first-time visitor has nothing to decide with. They cannot see what the tool produces, so the
only way to find out is to hand over an account first — which is the wrong order.

And the button provokes a question it does not answer: *why does a PDF generator want my Google
account?* Leaving that unanswered on the exact screen where it is asked is a strange choice, when
the answer is the best thing about the design (files are written to **the visitor's own** Drive,
so the app stores no credential and keeps nothing).

## Put the product on the gate

Three lines on what it makes, and a **live sample rendered through the same code path as the real
output** — not a screenshot, which drifts from the app the first time the layout changes.

Keeping the auth module ignorant of invoices is worth the one extra parameter:

```python
def require_login(L, preview=None) -> bool:
    ...
    if not signed_in:
        # Shown full width, below the sign-in box rather than beside it.
        if preview is not None:
            preview()
        return False
    return True
```

The caller passes a function that renders the sample; the gate never learns what an invoice is.
One of the three lines carries the permission answer, so the question dies where it is asked.

## The failure only other people see

The preview renders through pdf.js from a CDN. On my machine that has worked every single time,
which is exactly the problem: on a corporate network, or behind a content blocker, the script
never arrives and the visitor gets a **large empty grey box** on the one screen meant to invite
them in. It does not look like a blocked CDN. It looks like a broken app.

A script tag can fail three different ways, so all three route to one handler:

```html
<script>
  function pdfUnavailable() {
    const box = document.getElementById('pdf-container');
    if (box && !box.querySelector('canvas')) {
      box.innerHTML = '<div class="pdf-fallback">The preview could not load. '
                    + 'The PDF itself is fine — use the download button.</div>';
    }
  }
</script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"
        onerror="pdfUnavailable()"></script>
<script>
  if (typeof pdfjsLib === 'undefined') { pdfUnavailable(); }
  else {
    (async () => {
      try {
        /* … render pages … */
      } catch (err) { pdfUnavailable(); }
    })();
  }
</script>
```

Why all three:

- `onerror` catches a request that was refused outright.
- The `typeof` check catches a script that *loaded* but defined nothing — a proxy answering 200
  with an error page does this, and `onerror` never fires.
- `try/catch` catches a render that throws after the library loaded fine.

The `!box.querySelector('canvas')` guard matters too: a failure on page four should not wipe the
three pages that already drew.

## Takeaway

A login gate is a page, not a checkpoint — and it is the page with the highest traffic in any
gated app. It should say what is behind it, show it if that is cheap, and answer the question the
permission prompt raises.

And once something on that page depends on a third party, it has to announce its own failure.
You will never see that failure yourself: your network is, by definition, the one where it works.
