# RHTP Data Update Procedure

Follow these steps exactly when asked to "update the data." Before starting, review the **RHTPAnalyst Skill** rules in CLAUDE.md - especially Rule 1 (never hallucinate links), Rule 2 (use Playwright for bot-blocked sites), Rule 3 (State vs Vendor Applications), and Rule 5 (state-specific quirks).

## Step 1: Pull Current Data from Firebase and Create Backup

```bash
curl -s "https://rhtp-c8646-default-rtdb.firebaseio.com/states.json" > rhtp_results.json
```

Then immediately create a timestamped backup:

```bash
mkdir -p backups
cp rhtp_results.json "backups/rhtp_results-$(date +%Y-%m-%d-%H-%M).json"
```

Always pull from Firebase first - the web UI may have manual edits not in git.

## Step 2: Research State Updates

### What to check for each state
Visit each state's `programPageUrl` and search for:
1. **New RFAs/RFPs/NOFOs posted** - update `fundingOpportunityPosted`, `opportunityType`
2. **Application dates** - update `applicationOpenDate`, `applicationDeadline`
3. **Application links** - update `applicationLink`
4. **Vendor eligibility clarifications** - update `vendorEligible` (Yes or TBD)
5. **New program page URLs** - update `programPageUrl` if a better/dedicated page exists
6. **New contact emails** - update `rhtpEmail`
7. **Upcoming webinars and events** - actively search for webinars, town halls, stakeholder meetings, listening sessions, and info sessions with specific dates. These are high-priority for `nextSteps` because they often signal upcoming RFAs and let vendors engage early.
8. **Next steps / upcoming events** - update `nextSteps` array with dated items (webinars, deadlines, milestones)
9. **Any other material changes** - update `notes` with new findings

### Priority order
1. **Imminent states** - most likely to have new postings
2. **Planned states** - RFA/RFP Planned, Competitive Process Planned, etc.
3. **Notification states** - expecting notifications
4. **All remaining states** - check for any new activity

### Research methods
- Fetch each state's `programPageUrl` directly (use Playwright if bot-blocked - see CLAUDE.md Rule 2)
- **Check linked PDFs and presentation slides.** Key details (timelines, eligibility, funding amounts, application instructions) are often buried in PDF slide decks from stakeholder presentations, webinars, and town halls - not on the web page itself. Look for links to PDFs on the program page and sub-pages, and read through them. Example: Indiana's GROW statewide presentation PDF (found via in.gov/grow-rural-health/regional-grants/) contained detailed timeline and funding breakdowns not shown on the main page.
- Web search: `"[state] rural health transformation program" RFA OR RFP OR application 2026`
- Web search for events: `"[state] rural health transformation" webinar OR "town hall" OR "stakeholder" OR "listening session" OR "info session" 2026`
- Check state procurement portals if referenced in notes
- Check for updates on CMS.gov and SHVS (statehealthvaluestrategies.com)
- Run research in batches of 8-10 states simultaneously

## Step 3: Update the JSON Data

For each state with changes:
1. Update the relevant fields in `rhtp_results.json`
2. Set `lastUpdated` to today's date (ISO format, e.g., `"2026-02-07"`)
3. Update the footer date in `index.html` (line ~593)
4. Append key new findings to `notes` or rewrite if substantially changed
5. Update `nextSteps` array with any new dated items (format: `{"date": "YYYY-MM-DD", "label": "Description"}`)
6. Do NOT change fields that haven't changed

### Status field values
Use these exact values for `fundingOpportunityPosted`:
- `"Yes"` - RFA/RFP/application is live and accepting submissions
- `"Active (Rolling)"` - rolling/ongoing application process
- `"Reopened"` - previously closed, now accepting again
- `"Forthcoming"` - officially announced as coming soon
- `"Closed"` - was open, now closed
- `"No - RFA/RFP Planned"` - state has confirmed a competitive process is being developed
- `"No - Competitive Process Planned"` - competitive bidding/grants confirmed but no RFA yet
- `"No - Interest Form Available"` - interest/intent form available but not a full application
- `"No - Notifications Expected"` - state said notifications are coming by a specific date
- `"No"` - no downstream opportunity information available

## Step 4: Present Changes for Review

Before pushing, present a summary table:

```
| State | Field Changed | Old Value | New Value |
|-------|--------------|-----------|-----------|
| Alaska | fundingOpportunityPosted | Imminent | Yes |
| Alaska | applicationLink | (empty) | https://... |
```

Also note: total states checked, total with changes, any unreachable program pages.

**Wait for user approval before proceeding to Step 5.**

## Step 5: Push to Firebase (after approval)

```bash
curl -X PUT "https://rhtp-c8646-default-rtdb.firebaseio.com/states.json" \
  -H "Content-Type: application/json" \
  -d @rhtp_results.json
```

Verify the response contains the full data and record count matches 50.

## Step 6: Commit and Push to Git

```bash
git add rhtp_results.json index.html
git commit -m "Update RHTP data: [brief summary of key changes]"
git push origin main
git push vivo main
```

## Quick Reference: Partial Updates

- **"Check hot states"** = research only states where `fundingOpportunityPosted` is NOT `"No"`
- **"Check [state name]"** = research only the named state(s)
- Same procedure otherwise: pull from Firebase first, update, present changes, push
