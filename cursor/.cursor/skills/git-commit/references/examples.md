# Example commit messages

These come from a different project. Read them for shape and voice: how the
subject names behaviour, how the body moves from old behaviour to new, how
hashes, PR numbers, paths and measurements are cited. Do not reuse their
details.

## Subject only

```
Lock expired guest documents before deleting their files
```

## Short body: what changes and what the operator should know

```
Read today's upload count from the slot counter everywhere

The upload allowance check, the billing usage card and the file-scan precheck
all read users.daily_uploads now, so the number a page shows is the number the
server enforces and a text rescan of an uploaded file no longer frees a slot.
The migration backfills the counter from today's uploads so nobody gets a fresh
allowance on deploy day.
```

## Bug fix citing the commit that introduced it

```
Read the row before trusting a release that answered after a throw

7ef5bd8 retried the release of a scan whose insert lost its reply, but
a release can commit and lose its own reply, and the retry then finds
nothing to claim and answers false, which the catch took for "no row"
and refunded a second time; with a known row it also skipped the purge.

A false is now trusted only from an attempt no throw preceded. After a
thrown attempt the row itself decides: no row means the insert never
landed and the reservation is refunded directly; a row that can no
longer be claimed means the release landed, and it is purged; a row
still scanning with its reservation is left to the rescue and the cron.
```

## Performance change with scope limits and a measurement

```
Answer a signed-out get-session in the Worker entry

Every dashboard page asks /app/api/auth/get-session who is signed in. A
visitor who has not scanned yet has no session cookie, and better-auth
answers null before reading anything (no signed session_token, return null
in api/routes/session.mjs), yet each of those calls booted Next.

The entry now answers a GET of that exact path whose cookie header has no
better-auth.session_token with what Next sends for it: 200, null,
Content-Type application/json, Cache-Control no-store, Pragma no-cache and
Next's Vary, and no cookies. A request with a session token, any other
method (HEAD is Next's 404, POST better-auth's 415) and the trailing-slash
path still go to Next.

First request in wrangler dev, median of 3: 365 -> 26 ms.
```
