# SF Show Tracker — weekly update instructions

This repo publishes a single page, `index.html`, via GitHub Pages. It's a live listing of upcoming
SF venue shows (Venue / Artist / Genre / Date), styled as a "ticket stub" card grid. A GitHub Action
runs this file's instructions on a weekly schedule to refresh the data. Follow this process exactly.

## Critical: this runs non-interactively — do everything yourself, in this one turn

This process runs inside a single GitHub Actions job with no human present and no ability to be woken up
later. **Do not use the Task/Agent tool, and do not spawn background subagents to parallelize fetching the
8 venues.** A background agent's results only ever reach you if you're still running when it finishes and
you get notified — but the moment your final message is sent, the job's container is torn down immediately,
whether or not you actually finished the work. Spawning agents and then saying "I'll wait for them" ends
the run with nothing done and no error raised. Fetch each venue directly and sequentially yourself with
WebFetch, in the same turn, and finish by actually running the git commands below before your final message.

## Venues watched (8)

1. **Bill Graham Civic Auditorium** — https://billgrahamcivic.com/event-listing/ — scrapes cleanly (static HTML).
2. **The Regency Ballroom** — venue's own site (theregencyballroom.com/shows/) is JS-rendered, don't use it.
   Source from Bandsintown instead: https://www.bandsintown.com/v/10002004-the-regency-ballroom
3. **1015 Folsom** — https://1015.com/#calendar — scrapes cleanly (static HTML). Electronic/dance club — when
   a DJ night's genre is unclear, "Electronic / Dance" is a reasonable default for this venue specifically.
4. **The Warfield** — https://www.thewarfieldtheatre.com/events — scrapes cleanly (static HTML; has a "Load
   More" button, so only what loads by default is available without deeper pagination handling).
5. **August Hall** — https://www.augusthallsf.com/calendar/ — scrapes cleanly (static HTML).
6. **The Independent SF** — venue's own site (theindependentsf.com) only surfaces a partial sample on a plain
   fetch. Source from Bandsintown instead: https://www.bandsintown.com/v/10001466-the-independent — double
   check same-date multi-act entries here in particular (see reschedule-staleness note below).
7. **The Fillmore** — tracked via its Live Nation page: https://www.livenation.com/venue/KovZpZAE6eeA/the-fillmore-events
   (not thefillmore.com directly). Live Nation publishes genre directly — trust its tag over inferring from
   scratch, unless it's obviously wrong (e.g. tagging a Folk/Americana act "Rock").
8. **The Midway SF** — venue's own site (themidwaysf.com/events/) is JS-rendered, don't use it.
   Source from Bandsintown instead: https://www.bandsintown.com/v/10019956-the-midway

When given a new venue URL to add: test it first — if it returns nothing, shows a "JavaScript required"
placeholder, or clearly lists far more shows than fit on one page, search `"<venue name>" site:bandsintown.com`
first (best full-coverage option), falling back to AXS/Ticketmaster/Eventbrite/DICE/Prekindle only if no
Bandsintown page exists.

## Bandsintown scraping caveats — read carefully

Bandsintown venue pages (`bandsintown.com/v/<id>-<slug>`) return the full multi-month calendar in one fetch,
unlike AXS (which paginates 10-20 events at a time with no accessible API). Use Bandsintown for the three
venues above marked "source from Bandsintown." Two known issues:

- **Year mislabeling**: when fetched via a markdown-converting fetch tool, Bandsintown's calendar headings
  can show the wrong (one-year-behind) year — e.g. real Sep 2026 dates grouped under a "September 2025"
  heading. Month/day/artist are correct; only the year label is wrong. Cross-check a couple of dates against
  another source (AXS, the venue's own site, or Live Nation) before trusting the year, and correct if needed.
- **Stale reschedules**: Bandsintown can list a show on its original date after it's actually been moved. If
  a pull shows two acts on the same date at the same venue, or anything else that looks off, verify against
  the venue's own site or Songkick/Ticketmaster for that specific date before including it — don't take a
  same-date double-booking at face value.
- **Cloudflare blocks from this environment**: when this runs inside GitHub Actions specifically, Bandsintown
  can return a Cloudflare "Sorry, you have been blocked" interstitial instead of the real page (GitHub's
  server IPs sometimes have a worse bot-detection reputation than other fetchers). If that happens: retry
  the fetch once, and if still blocked, fall back to the venue's own site or an AXS/Ticketmaster/Songkick
  search for that specific venue for this run rather than getting stuck — a venue with thinner data one week
  beats the whole run silently failing. Note in the commit message if a venue's primary source was blocked.

## Window rule

Include a show only if its date is **at least 2 weeks and at most 3 months** from today (the day this runs).
Recompute this window fresh each run — don't reuse a previous run's cutoff dates.

## Genre methodology

Most venues don't publish genre (Live Nation/The Fillmore is the exception). Infer genre from artist
knowledge; do a quick web search only when uncertain and the venue's show volume is small enough to make
that practical (don't burn a search per artist on a 60+ show venue like The Independent — mark `TBD` there
instead of guessing).

## Output: updating index.html

`index.html` is a single self-contained file. Near the bottom, inside a `<script>` tag, there's a
`const shows = [...]` array — each entry has `date`, `venue`, `artist`, `genre`, `note`, `status`
(`"onsale"` | `"soldout"` | `"presale"`) fields. Replace this array with the freshly scraped, filtered,
deduped show list, sorted by date then venue. Do not change the surrounding HTML/CSS/JS structure, the
venue-pill filter list, the color palette, or the per-card "Contact" button (the `bookingAgentInfoUrl()`
helper and the `.contact-link` element in each card) — only the data.

Also update the header's meta chips (Updated date, Window range, Venue count, Show count) to reflect the
new run.

## Committing

After rewriting `index.html`, commit and push directly to `main` yourself, in this same turn — do not end
your turn or your final message until this has actually run and you've confirmed it worked:

```
git add index.html
git commit -m "Weekly update: <today's date>"
git push
git log -1
```

Check the `git push` output for confirmation it reached `origin/main` (not an error), and check `git log -1`
shows your new commit. Only after confirming that should you send your final summary message. GitHub Pages
will pick up the change automatically once pushed — no further action needed beyond the push itself.
