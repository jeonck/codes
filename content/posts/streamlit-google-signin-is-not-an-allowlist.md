+++
title = "Google sign-in on a Streamlit app: st.login() is only half the gate"
date = 2026-09-21
summary = "A successful Google sign-in proves someone owns a Google account — not that they may use your app. Without an allowlist the page is open to everyone, and a new OAuth client answers every sign-in with 403 until the consent screen is dealt with."
tags = ["streamlit", "oauth", "google", "auth", "trial-and-error"]
+++

Notes from putting Google sign-in in front of a Streamlit tool that generates invoices — a form
people type bank account numbers into, so an open page was not an option.

## The easy mistake

Streamlit ships OIDC since 1.42, and the happy path is three lines:

```python
if not st.user.is_logged_in:
    st.button("Sign in", on_click=st.login, args=("google",))
    st.stop()
```

It is tempting to read `is_logged_in` as "this person may use the app". It doesn't say that.
It says Google confirmed they own *a* Google account — and everyone has one. Ship that and the
form is open to the entire internet, behind one extra click.

Authentication answers *who is this*. The app still has to answer *may they*:

```python
def allowed_emails() -> set:
    try:
        configured = st.secrets["app_auth"]["allowed_emails"]
    except Exception:
        return set()          # no list configured
    return {str(e).strip().lower() for e in configured if str(e).strip()}
```

Two things matter more than the lookup itself:

- **An empty list means locked, not open.** Missing config is a misconfiguration; treating it as
  a permissive default is how an app ends up unguarded in exactly the case where someone fumbled
  the secrets.
- **Check `email_verified`, not just `email`.** A provider can hand back an address it has not
  confirmed belongs to the person.

## The 403 nobody warns you about

With everything configured correctly, the first sign-in still fails:

```
403: access_denied
The app is currently being tested and only developer-approved testers can access it.
```

Nothing in the code can fix this. A new OAuth client starts with its consent screen in
**Testing**, where Google refuses everyone not on the test-user list. Two ways out, on
**APIs & Services → OAuth consent screen → Audience**:

- add the address under **Test users** — immediate, capped at 100 people, or
- press **Publish app**.

Publishing sounds like the scarier option and isn't. Verification review is only required for
sensitive scopes; `openid`, `email`, `profile` and `drive.file` are all non-sensitive, so the
app publishes immediately. And publishing does not widen who can get in — the allowlist still
decides. That is the payoff of not leaning on Google's gate: the consent screen's audience
setting becomes a deployment detail rather than a security control.

## The order that works

1. Create the OAuth client (**Web application**) and register the redirect URI for every place
   the app runs. The path is `/oauth2callback`, and scheme, port and trailing slash must match
   exactly or you get `redirect_uri_mismatch`.

   ```
   http://localhost:8501/oauth2callback
   https://<app>.streamlit.app/oauth2callback
   ```

2. Settle the consent screen's audience *before* testing sign-in, so a 403 doesn't get
   misread as a config error.

3. Write the secrets. Note the app's own settings live outside `[auth]`, which belongs to
   Streamlit — `[auth.<name>]` is read as an identity provider, so an `[auth.users]` of your own
   would be taken for one.

   ```toml
   [auth]
   redirect_uri = "https://<app>.streamlit.app/oauth2callback"
   cookie_secret = "<random>"

   [auth.google]
   client_id = "<...>.apps.googleusercontent.com"
   client_secret = "<...>"
   server_metadata_url = "https://accounts.google.com/.well-known/openid-configuration"

   [app_auth]
   allowed_emails = ["you@example.com"]
   ```

4. Verify the *refusal* path, not just the success path: sign in with an address that is not on
   the list and confirm the app body never renders.

## Two settings that fail quietly

**`streamlit`, not `streamlit[auth]`.** Sign-in dies with `No module named 'httpx'` the moment
someone clicks the button. Streamlit's OIDC imports `authlib.integrations.starlette_client`,
which needs httpx — and Authlib does not depend on httpx itself. Streamlit declares both under
its own extra, so depend on that rather than pinning Authlib and drifting from it:

```
streamlit[auth]>=1.42
```

**A scope added after the first sign-in does nothing.** Widening `client_kwargs.scope` later —
to reach Drive, say — leaves existing sessions holding a cookie whose token was issued for the
old scopes. The symptom is a feature that silently refuses to work for the one person who was
already signed in: you. Sign out, sign back in.

## What the gate is actually worth

Once identity is established properly it can be spent. Streamlit will expose the provider's
access token (`expose_tokens = ["access"]`), so the app can call Google APIs **as the signed-in
person** — files written to their own Drive, under their own account, revocable by them from
their Google settings, with no long-lived credential stored by the app at all.

That is the part worth keeping: the difference between an app that holds a key to everyone's
data and an app that holds nothing and borrows permission per person. The allowlist is what
makes the first half of that sentence true.
