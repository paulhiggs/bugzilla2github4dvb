# bugzilla2github4dvb

[paulhiggs/bugzilla2github4dvb](https://github.com/paulhiggs/bugzilla2github4dvb)
is a fork of
[berestovskyy/bugzilla2github](https://github.com/berestovskyy/bugzilla2github).
Its `forDVB` branch adapts the tool to migrate the DVB Bugzilla (`bugzilla.dvb.org`) to GitHub Issues. It is used
with XML exports from **Bugzilla version 5.2+**: the export's root element
reads `<bugzilla version="5.2+" ...>`. It has not been tested with exports from
other Bugzilla versions.

The changes are listed below against the original project (up to commit
`705e215`). See [README.md](README.md) for the original documentation.

## Usage

```sh
# Dry run: reads GitHub, changes nothing, prints all the warnings
python bugzilla2github -x export.xml -o OWNER -r REPO -t TOKEN

# Real run: creates labels, milestones and issues
python bugzilla2github -x export.xml -o OWNER -r REPO -t TOKEN -f
```

Requires Python 3 and `requests`.

The token needs **Issues: Read and write** on the target repository
(fine-grained token) or the `repo` scope (classic token). The beta import API
option is `-e`, not `-b` as the original README says.

A real run can be stopped and started again with the same command: issues
already on GitHub and up to date are skipped, and the run continues from the
first issue that is missing or different.

## Fixes

- **GitHub authentication.** GitHub no longer accepts the token as an
  `access_token` URL parameter, so every request was anonymous, limited to 60
  per hour, and failed. The token is now sent in the `Authorization` header.
- **"Not found" vs. failure.** Any failed request used to count as "this
  issue does not exist". Only a 404 does now, and other failures stop the
  script.
- **Updates use `PATCH`**, as the GitHub API requires.
- **Attachments referenced more than once, or missing from the export.**
  These used to crash the script with `KeyError`. Now:
  - an attachment can belong to several comments;
  - a missing attachment shows a placeholder and a warning.
- **Empty `<creation_ts>`.** Some Bugzilla 5.2 exports leave it empty, which
  crashed the script. The date of the bug's first comment is used instead.
- **Attributes of repeated XML elements.** Attributes such as
  `isobsolete` on `<attachment>` overwrote each other at bug level. They are
  now kept with each element.
- **Label names.** Label names are now:
  - URL-encoded when checked on GitHub, so names like `dvb.org/metadata/`
    work;
  - stripped of commas, which GitHub rejects;
  - cut to GitHub's 50-character limit;
  - de-duplicated without regard to case, as GitHub treats labels.
- **Python 3 only.** The Python 2 code (`six`, `reload(sys)`) is removed.
  Mixed tab/space indentation and an invalid `"\d"` escape are fixed.
- **Usage text.** The beta API option is shown as `-e`, which is what the
  script accepts.

## Changes

### No Bugzilla information is dropped

The original script printed a warning and dropped any field it did not map.
Now every field is kept:

- **Labels.** Each of these becomes a GitHub label, using its own name when the
  conversion tables have no entry for it:
  - product
  - component
  - priority (`P1` to `P5`)
  - each keyword
  - resolution
- **Bugzilla fields table.** Each issue description has a table of the exact
  values: product, component, version, target milestone, status and
  resolution, severity, priority, keywords, URL (`bug_file_loc`), whiteboard,
  deadline, hardware, OS, time tracking, restricted groups and flags.
  Placeholder values such as `---`, `unspecified` and `Not used` are left out.
- **Attachments.** Each attachment also shows who attached it, when, when it
  was last modified, and whether it is a patch or obsolete. The XML export does
  not contain the attachment files, and the issue says so.

Only Bugzilla internals (tokens, access settings, sort order) are ignored.

### GitHub milestones

Each Bugzilla target milestone becomes a GitHub milestone, and each issue is
added to its milestone.

- A milestone is created closed if all its bugs are closed, and open
  otherwise.
- An existing milestone with the same title is reused.

### Text shown as written

Bugzilla text is plain text, but GitHub reads issue text as Markdown and HTML.
Comment text, attachment descriptions and field values are now escaped, so
that:

- XML such as `<ServiceList version="1">` is shown, not hidden as HTML;
- lines starting with `#`, `=`, `-` or `1)` are not turned into headings,
  rules or lists;
- `*`, `_`, `~` and backticks are not turned into formatting;
- indentation is kept without turning text into code blocks;
- `@schemeIdUri` and similar words are shown as code, so they do not link to
  or notify GitHub users;
- URLs are still links.

This was checked with GitHub's Markdown API: 310 real DVB comments rendered
identical to the original text.

### People and email addresses

People with no GitHub login in `EMAIL2LOGIN` are shown by the part of their
email address before the `@`, with their Bugzilla name when there is one, for
example `Patrick Ladd (pladd)`. Full addresses are no longer published in the
issue header, CC list, attachments or flags.

No `@` is added in front of these names, because `@name` would link to and
notify whichever GitHub user has that login. Email addresses written inside
comment text are not changed.

### Rate limits and resuming

- **Retries.** Every GitHub request is retried after network errors and
  server errors, up to 10 times with increasing waits.
- **Rate limits.** When GitHub reports a rate limit, the script waits as long
  as GitHub says. If GitHub gives no time, it waits a minute longer each time,
  up to 15 minutes, until the request is accepted. Rate-limit waits do not
  count as failures.
- **Pacing.** The script pauses one second after each request that changes
  GitHub, as GitHub asks.
- **Resuming.** A resumed run skips issues already up to date: same title,
  body, state, labels, assignees and milestone. It also skips placeholders for
  deleted bugs that are already closed. This saves GitHub's limit of about 500
  content changes per hour.

GitHub's limit means a real run creates about 250 issues per hour, because
creating a closed issue takes two requests: create, then close. About 3,400
DVB bugs take roughly 12–14 hours.

## Unchanged from the original

- **Open/closed state** uses the original `STATUS2STATE` table. In
  particular, `VERIFIED` bugs become **open** issues. The resolution is kept as
  a label and in the fields table.
- **Issue numbers** match Bugzilla bug numbers. Deleted bug numbers get closed
  placeholder issues.
- **Comments.** In the default (stable API) mode, all comments go into the
  issue description, and issues are dated by the GitHub run, not by Bugzilla.
  The original dates are shown in the text.
- **Bug references.** `bug 123` in text becomes `bug #123`, which links to
  that issue. `comment #3` in Bugzilla text also links to issue #3, not to a
  comment.
