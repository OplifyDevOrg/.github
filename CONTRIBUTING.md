# Contributing

How work happens here. It applies to both repositories.

If you are new: read this once, then `make up` in the API repo. Everything below
is convention rather than ceremony — where a rule exists, the reason it exists is
given, because a rule you understand is one you can tell when to break.

---

## The loop

1. **An issue first.** It is where the reasoning lives, so the record survives
   after the chat that produced it is gone.
2. **A branch** named `<type>/<issue>-<slug>` — `fix/27-outbound-media`,
   `feat/31-provision-staging`. Types: `feat`, `fix`, `refactor`, `docs`,
   `chore`, `ci`, `test`.
3. **A pull request** into `main`, whose body says `Closes #27`.
4. **Close the issue** when it merges. See the note under _Branches_ — GitHub
   does not always do this for you.

Skip the issue only for something genuinely trivial: a typo, a comment, a
formatting fix. When unsure, open one — a small issue costs nothing.

Add the issue to the [Oplify Delivery](https://github.com/orgs/OplifyDevOrg/projects/1)
board and set **Sprint** and **Priority**. Leave **Status** alone; it follows the
issue and its PR on its own.

```bash
gh project item-add 1 --owner OplifyDevOrg --url <issue-url>
```

Set **Blocked on** to a person when the work needs access nobody else has — a
cloud dashboard, DNS, a key. That is the difference between work that looks
stalled and work that is waiting on someone specific.

---

## Writing an issue

Say what breaks, for whom, and how you know. Someone reading in six months needs
the evidence, not the conclusion.

- **Lead with the observable symptom**, not your theory. "Attaching a file and
  sending it does not deliver" before any mention of presigned URLs.
- **Give the evidence** — file and line, the log, the failing command.
- **Name what you ruled out.** "Receiving works — verified against a real inbound
  message" stops the next person re-checking it.
- **Say how it will be verified.** If the real test is a round trip through the
  UI, say so; a unit test that cannot catch the bug is worse than none.

---

## Branches

`main` is the trunk and it deploys production. Branch from it, PR back into it.

Never commit to `main` directly. A merged PR is the only way in, because the
gates below run on PRs.

> **Closing issues.** `Closes #N` only auto-closes when the PR merges into the
> repository's _default_ branch. That is `main` here, so it works — but if you
> ever target another branch, close the issue by hand.

---

## Commits

Conventional prefixes — `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`, `ci:`,
`test:` — optionally scoped: `fix(auth): …`.

**The subject says what changed. The body says why.** The why is the part that is
expensive to recover later:

```
fix(auth): give refresh tokens a jti so same-second logins do not collide

Two sign-ins for the same user within one second produced a byte-identical
refresh token — same payload, and `iat` has whole-second resolution — so the
second Session.create violated the unique constraint and returned 500. A
double-clicked sign-in button hits this.
```

If a change is not obvious from the diff, the body is where you explain it. If
you discovered something surprising, write it down — that is the sentence that
saves someone a day.

---

## Pull requests

Say what changed, why, and **how you verified it**. "Tests pass" is not
verification if the tests could not have caught the bug.

Call out anything that needs a deploy step, a secret, or a manual action. Those
are the things that get lost between merge and production.

Keep PRs to one idea. If you fix something unrelated on the way, that is a
second PR — reviewers cannot review two things at once, so they review neither.

### Reviewing

Look for, roughly in order:

1. **Correctness** — does it do what it claims, including the edge the issue described?
2. **Blast radius** — what else touches this? Multi-tenancy in particular: every
   tenant-scoped query must filter on `organizationId`, and a missing filter is a
   cross-tenant data leak.
3. **Failure modes** — what happens when the network is down, the token is
   expired, the upload is half-written? Silent fallbacks are worse than errors.
4. **Whether it can be verified** — if you cannot tell how the author knows it
   works, ask.

Approving is a statement that you would be comfortable being paged for it.

---

## The gates

Every PR runs CI. Nothing merges red.

| Gate                      | What it actually proves                                                       |
| ------------------------- | ----------------------------------------------------------------------------- |
| Lint & format             | Style is mechanical, so it never needs discussing in review                   |
| Tests                     | The suite runs against **real Postgres and Redis**, not mocks                 |
| Docker build & smoke test | The image boots, migrations apply from inside it, all three entrypoints start |

The API's test suite is deliberately run via `make test`, which provisions a
scratch database. Running `npm test` directly makes the integration tests
**skip themselves** and report green having tested nothing.

The frontend's lint gate is a **ratchet, not a clean gate**: the tree carries a
backlog of errors, so it fails only if your PR _adds_ to it. Lower the baseline
when you burn some down.

---

## Environments

| Environment | Runs on               | Fed by      | Serves the SPA                |
| ----------- | --------------------- | ----------- | ----------------------------- |
| Local       | Docker on your laptop | your branch | Caddy, same origin as the API |
| Staging     | Oracle Cloud          | `main`      | Caddy, same origin as the API |
| Production  | AWS EC2               | `main`      | nginx, same origin as the API |

**Same origin everywhere** is deliberate. The dashboard and the API share a
hostname, so `VITE_API_BASE_URL` is the relative `/api/v1`, CORS does not apply
to the dashboard at all, and a mixed-content or cross-site-cookie failure is
structurally impossible rather than guarded against.

Nothing escapes staging: mail goes to Mailpit and is never delivered, the
WhatsApp test number reaches only allow-listed recipients, and
`WEBHOOK_SIDE_EFFECTS=false` suppresses AI replies, CRM and Sheets writes.

### Running locally

```bash
cd oplify-messaging-api
make up          # everything: app, API, workers, database, HTTP and HTTPS
make doctor      # checks the whole setup, including your WhatsApp token
```

`make` on its own lists every command. Docker and `make` are all you need to run
it; Node on the host is needed only for `make doctor` and `make test`.

Deeper references live in the API repo: `docs/LOCAL_DEVELOPMENT.md`,
`docs/STAGING.md`, `docs/DEPLOYMENT.md`, and `docs/WHATSAPP.md` for tokens,
webhooks and Meta error codes.

---

## Deploying

Merging to `main` deploys. Production goes through three gates: proven in CI,
proven against real production Postgres and Redis _before_ the swap, then
verified through nginx afterwards asserting the running revision plus database
and Redis health. A failed gate rolls back automatically.

`docs/DEPLOYMENT.md` has the runbook, the rollback, and the list of things never
to run on the box. Read it before your first deploy, not during.

---

## Secrets

Never commit one. Local development needs no shared secret at all — `make up`
generates yours. The handful genuinely shared across the team are committed
**encrypted** with dotenvx; you need `.env.keys` from the password manager to
decrypt them, and only for `make webhooks`.

Production and staging keep their own uncommitted `.env` on their own boxes.

If you ever paste a secret somewhere it should not be — a screenshot, a chat, a
log — rotate it. It is cheap, and assuming otherwise is how leaks persist.
