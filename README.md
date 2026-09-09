# fang-and-bone (`fnb`)

<p align="center"><img src="assets/fang-and-bone-icon.png" alt="Fang & Bone: a purple monster with ivory horns, fangs, and a bone charm" width="240" height="240"></p>

App Store Connect user and TestFlight maintenance from your terminal.

This repo is the public face of `fnb` — downloads, release notes, and this
page. The source is private.

### Why "Fang and Bone"?

In *Breath of the Wild* and *Tears of the Kingdom*, Kilton runs a shop by
that name — a cart that only turns up after dark, buying monster parts no
other merchant in Hyrule will touch, one piece at a time, for a currency no
other shop accepts. That's the shape of this problem, too: App Store
Connect access is made of oddly-specific parts — a team role here, a
per-app visibility grant there, a TestFlight group membership somewhere
else — and there is no single screen in Apple's UI that trades in all of
them at once. `fnb` is the shop that does: hand it a person, and it deals in
exactly the parts that make up their access, onboard or off, no more and no
less.

Adding a new hire to TestFlight across several apps is a multi-day, multi-step
chore in the App Store Connect web UI, and the steps have to happen in order:

1. Invite them to the App Store Connect team with a role and app visibility.
2. Wait for them to accept — **Apple expires the invitation after 3 days.**
3. Add them as a TestFlight tester in each app's internal beta group.

Step 3 cannot happen before step 2. Apple enforces that server-side. So:

```console
$ fnb onboard jane@example.com --as ios-engineer

Onboarding jane@example.com
  ● invite jane@example.com to the team as DEVELOPER with access to 2 app(s)
  ○ jane@example.com has not accepted their team invitation yet

  Next: jane@example.com accepts the email from Apple, then re-run this command.
  Check progress any time with: fnb status jane@example.com
```

Then, once she accepts, **the same command finishes the job**:

```console
$ fnb onboard jane@example.com --as ios-engineer

Onboarding jane@example.com
  ● grant visibility of 2 app(s)
  ● add jane@example.com as a TestFlight tester in 2 group(s)
  – TestFlight emails the tester automatically once a build is available to their group

  ✓ jane@example.com is set up
```

Running it repeatedly is safe. It reads live state each time and only does what
is still missing, so a fully onboarded person produces an empty plan.

## Install

```console
brew install jason/tap/fnb
```

Homebrew 6 added tap trust, so the fully-qualified name matters —
`brew tap` followed by `brew install fnb` will fail.

## Getting started

```console
fnb auth login     # guided; stores the key in your keychain
fnb doctor         # verifies credentials, permissions, and presets
```

`fnb auth login` opens the right App Store Connect page, watches `~/Downloads`
for the `AuthKey_*.p8` you download, reads the Key ID out of its filename, asks
only for the Issuer ID, verifies the key against the live API, and offers to
shred the downloaded file.

You need a **Team key with the Admin role**. Apple provides no API for creating
API keys — that is the root of trust, so it is web UI only — and your Account
Holder must have requested App Store Connect API access once, ever, before any
key can exist.

## Config

Presets turn a role and a list of apps into one word. Put this in
`~/.config/fnb/fnb.toml`, or in `./fnb.toml` to commit it alongside a project:

```toml
[preset.ios-engineer]
role = "DEVELOPER"
apps = ["acme", "acme-qa"]

[preset.product]
role = "MARKETING"
apps = ["acme"]

[alias]
acme = "com.example.Acme"
acme-qa = "com.example.AcmeQA"
```

Point engineers at your QA builds rather than production, and leave the
production alias defined but out of the default preset — granting it then
takes a deliberate one-off rather than happening by accident.

Aliases map to **bundle IDs** on purpose: Apple's name filter is exact-match
only and display names change, while bundle IDs do not.

Config is validated on load, so mistakes surface immediately rather than as a
confusing API error later. A preset pairing `ADMIN` or `FINANCE` with an `apps`
list is rejected, because Apple gives those roles access to every app and
silently ignores the scoping.

## Commands

```
fnb onboard <email> --as <preset>     the flagship — re-runnable, resumes
fnb offboard <email>                  mirror image; confirms first
fnb status <email>                    one person, everything about them
fnb doctor                            check credentials, permissions, config

fnb auth      login | status | logout | token
fnb apps      list
fnb users     list
fnb invites   list | cancel
fnb groups    list
fnb testers   list | add | remove
fnb config    list | path
fnb api       <path>                  raw authenticated passthrough
```

Every step `onboard` performs is also available as its own command, for when you
want one specific change and nothing else.

### Useful flags

- `--dry-run` — print the plan and change nothing.
- `--json` — machine-readable output on stdout, with sorted keys so diffs are
  stable. Progress is suppressed, so `fnb ... --json | jq` is clean.
- `-y, --yes` — skip confirmations. Required for destructive actions in scripts:
  a non-TTY without `--yes` is a **hard error**, never an assumed yes.
- `--create-groups` — create an internal beta group for apps that lack one.
  Off by default, since silently creating groups in your account is a surprise.
- `--allow-role-change` — permit raising an existing member's role so they
  become eligible to test.

### Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | error |
| 2 | cancelled |
| 4 | authentication required or insufficient key permissions |
| 64 | bad usage |

## Mac app

The same engine, with a window — a SwiftUI app for macOS 26 that imports the
core library directly. It never shells out to `fnb`, so the two agree by
construction: the app renders the same plan the CLI prints.

What it does, screen by screen: **Onboard** and **Offboard** preview a plan
before running it, show live progress per step, and ask before anything that
changes permissions. Onboard can also watch for the invitation to be accepted
and finish the job automatically — the `--wait` mode the CLI never got.
**Look Up** is `fnb status` across every app in the account. **Apps**, **Team**,
**Invitations** and **Testers** are the inspect commands as sortable tables
with the obvious actions in context menus. **Doctor** runs the same checks as
`fnb doctor`. **Presets** edits the config natively and writes the same TOML,
and can link the CLI's own `fnb.toml` so both share one file. **API Console**
is `fnb api`. **Activity** records every write the app made, which the CLI
cannot do.

Sign-in mirrors `fnb auth login`: it opens the right App Store Connect page,
watches Downloads for the `.p8`, reads the Key ID from the filename, verifies
the key against the live API before storing it, and shreds the download.

The app is hardened, and deliberately *not* sandboxed: the in-app updater
replaces the app's own bundle in `/Applications` and relaunches it, which no
container can reach. Everything else is written as if the sandbox were still on
— security-scoped bookmarks, no path assumptions — and every entitlement is
justified in the project file. The app and the CLI keep **separate** keychain
items; the sign-in sheet offers to adopt the CLI's key so you only download one.

## Design notes

Some Apple behavior is non-obvious enough to be worth stating, since it shaped
the tool:

- **Internal testers are `betaTester` records, not team users.** There is no
  link from a `users` record to a beta group. What makes a tester *internal* is
  the group's `isInternalGroup` flag, and Apple enforces the
  "must be an accepted team member" rule as a rejection at link time — an
  undocumented `409 STATE_ERROR`. `fnb` plans around it and translates it into
  plain English if it fires anyway.
- **There is no resend endpoint for team invitations.** Re-inviting is a delete
  followed by a fresh create, which is what `fnb` does for an expired invite.
- **Adding a tester to a group auto-emails them** — but only once a build is
  available to that group. `fnb` says so rather than implying mail went out.
- **Apple sends no `Retry-After`**, even on 429, so rate limits get exponential
  backoff with jitter.
- **Offboarding is not instant.** Apple caches access for up to 10 minutes.

The CLI keeps credentials in the macOS login keychain via `/usr/bin/security`,
the same approach `gh` takes. This is deliberate: a keychain item's ACL records
the signing identity of whichever binary created it, and an ad-hoc-signed
binary's identity changes on every rebuild — as does a Homebrew binary's on
every upgrade — which revokes the grant and re-prompts forever. Letting Apple's
own stable-identity tool own the item avoids that entirely. The trade, stated
plainly: confidentiality becomes user-level rather than per-application, since
any process running as you can read the key by invoking `security` itself.
