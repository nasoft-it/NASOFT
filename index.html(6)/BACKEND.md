# NASOFT — backend integration

The site runs as a single self-contained `index.html`. Everything below concerns
turning the demo layer into a real deployment.

## 1. Configuration

All placeholders live in one object near the top of the inline script:

```js
var CONFIG = {
  NASOFT_DOMAIN:   'nasoft.it',
  NASOFT_EMAIL:    '',   // [ADD_LATER]
  WHATSAPP_NUMBER: '',   // [ADD_LATER] digits only, incl. country code
  INSTAGRAM_URL:   '',   // [ADD_LATER]
  NAC_WEBSITE_URL: '',   // [ADD_LATER]
  AUTH_PROVIDER:   '',   // [ADD_LATER]
  API_BASE:        '/api'
};
```

Anything left empty renders its control in a visibly disabled state with a
tooltip naming the missing variable. Nothing is faked.

`N8N_WEBHOOK_URL` and `DATABASE_URL` are deliberately **not** in this file.
They are server-side environment variables only.

## 2. Replacing the demo auth

`Auth` is an adapter object. Implement these methods against your real provider
(Supabase, Auth0, Clerk, or your own API) and assign it:

| Method | Returns |
|---|---|
| `currentUser()` | user object or `null` (synchronous, from session) |
| `register({name, email, password, phone})` | `{user_id, email}` |
| `verifyEmail(token)` | `true` |
| `resendVerification(email)` | `null` |
| `login(email, password)` | user object |
| `logout()` | `true` |
| `requestPasswordReset(email)` | `null` — identical response whether or not the account exists |
| `resetPassword(token, password)` | `true` |
| `updateProfile({name, phone})` | user object |
| `requestEmailChange(newEmail)` | `null` |
| `changePassword(current, next)` | `true` |

Then set `CONFIG.AUTH_PROVIDER` to any non-empty string — that alone removes the
demo banner and the demo-only verification links.

Failure convention: throw `Error(message)`; the message is shown to the user
verbatim, so write it for a person. `login` throws with `err.code =
'EMAIL_UNVERIFIED'` when the account exists but is unverified.

## 3. Replacing the demo data layer

`Data` is the second adapter: `list()`, `get(id)`, `create(input)`,
`respond(id, message)`. Note the rule the demo already follows and your server
must enforce: **no method takes a caller-supplied `user_id`.** Identity comes
from the session. A record belonging to another user returns not-found, never
"forbidden but exists" — that distinction leaks existence.

## 4. Server routes

The client calls your server; your server calls n8n.

```
POST /api/submit-problem
POST /api/submit-idea
POST /api/contact
POST /api/status-response
```

Each must verify the session server-side, attach `user_id` / `verified_email` /
`name` from it, then forward to `N8N_WEBHOOK_URL`. The webhook URL must never
reach the browser.

## 5. Data model

Implemented as specified: `users`, `submissions`, `status_history`, `messages`.
Submissions inherit `user_id`, `submitter_email`, and `submitter_name` from the
authenticated account — the submission form has no email field.

## 6. Routing

Hash routing (`#/submit`) so the file works from any static host with no server
config. To switch to clean paths (`/submit`), replace `parseHash()` with
`location.pathname`, swap `hashchange` for `popstate`, intercept link clicks
with `history.pushState`, and add a server rewrite of all paths to `index.html`.
`sitemap.xml` already lists the clean paths.

## 7. Attachments

The form validates type and size client-side (10 MB each, 5 files max) and
records filenames. Actual upload needs a storage bucket and a signed-URL
endpoint — wire it into `Data.create` before the record is written.

## 8. Not included, by design

No pricing. No fabricated statistics, testimonials, clients, awards, offices, or
partnerships. No accreditation claim for NAC — the education page states plainly
that none exists. `privacy.html` and `terms.html` content is a working draft and
needs legal review before launch.
