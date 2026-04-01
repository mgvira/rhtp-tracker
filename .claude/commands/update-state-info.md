# /update-state-info

Research all 50 states for RHTP vendor/subaward application updates and push changes to Firebase.

## Before You Start

1. **Read the RHTPAnalyst Skill rules** in the project's CLAUDE.md file - especially:
   - Rule 1: NEVER hallucinate links. Only use URLs you have verified exist.
   - Rule 2: Use Playwright for bot-blocked sites (Ohio, Minnesota, Vermont, NH, MI, NY, KY).
   - Rule 3: We are tracking **Vendor Applications** (RFA/RFP/NOFO from states to vendors), NOT State Applications (to CMS - those are done).
   - Rule 4: Check PDFs and presentation slides linked from program pages - key details are often buried there.
   - Rule 5: Always update `lastUpdated` dates on changed records.
   - Rule 6: State-specific quirks (Minnesota domain, Virginia DMAS, Delaware placeholder, Ohio solicitation page, etc.).
   - Rule 7: Data integrity - pull from Firebase first, never skip review, preserve existing data, back up first.

2. **Read UpdateData.md** for the full step-by-step procedure.

## Step 1: Pull Data and Create Backup

```bash
# Pull latest data from Firebase (may have manual edits from web UI)
curl -s "https://rhtp-c8646-default-rtdb.firebaseio.com/states.json" > rhtp_results.json

# Create timestamped backup
mkdir -p backups
cp rhtp_results.json "backups/rhtp_results-$(date +%Y-%m-%d-%H-%M).json"
```

Confirm backup was created before proceeding.

## Step 2: Research All 50 States via Parallel Subagents

Sort the 50 states by priority, then spawn **5 parallel `general-purpose` subagents** (10 states each):

### Priority grouping:
1. **Agent 1 (Hottest):** States with `fundingOpportunityPosted` = "Yes", "Active (Rolling)", "Reopened", or "Forthcoming"
2. **Agent 2:** States with "Planned" or "Competitive Process Planned" or "Notifications Expected"
3. **Agent 3:** Next batch of "No" states (alphabetical A-K)
4. **Agent 4:** Next batch of "No" states (alphabetical L-O)
5. **Agent 5:** Remaining "No" states (alphabetical P-Z)

If a priority group has fewer than 10 states, fill with states from the next group. Each agent should handle exactly 10 states to balance the workload.

### Each agent's prompt MUST include:

1. **The full current JSON data** for its 10 assigned states (copy-paste the records).

2. **These critical rules:**
   - We track VENDOR APPLICATIONS (RFA/RFP/NOFO from states to vendors/providers). NOT the CMS application each state submitted - those are done.
   - A "Project Narrative" or "CMS Application" = State Application (ignore). An "RFA," "RFP," "NOFO," or "Grant Application" directed at providers/vendors = Vendor Application (this is what we want).
   - NEVER invent or guess URLs. Only use URLs you have verified exist via WebFetch or web search.
   - If WebFetch returns 403/404, the site may be bot-blocked. Try verifying via Google search before concluding it's broken. Known bot-blocked: Ohio, Minnesota, Vermont, NH, MI, NY, KY.
   - Check for PDF links (presentations, slide decks, FAQ docs) on program pages and sub-pages - key details are often in PDFs, not on the HTML page.

3. **Research instructions for each state:**
   - Fetch the state's `programPageUrl` via WebFetch
   - Web search: `"[state name] rural health transformation program" RFA OR RFP OR application 2026`
   - Web search for events: `"[state name] rural health transformation" webinar OR "town hall" OR "stakeholder" OR "listening session" OR "info session" 2026`
   - Check state procurement portals if mentioned in existing notes
   - Look for: new RFAs/RFPs/NOFOs posted, application dates, application links, vendor eligibility info, new contact emails, timeline changes
   - **Actively search for upcoming webinars, town halls, stakeholder meetings, listening sessions, and info sessions.** These are often where states announce timelines, answer vendor questions, and signal upcoming RFAs. Add any with specific dates to `nextSteps`.
   - Look for linked PDFs (stakeholder presentations, webinar slides, town hall decks) and check their content
   - Look for other upcoming events with specific dates (deadlines, milestones) to populate `nextSteps`

4. **Output format - return a JSON array:**
   ```json
   [
     {
       "state": "StateName",
       "changes": {
         "fieldName": "new value",
         "notes": "Updated notes text...",
         "nextSteps": [{"date": "2026-04-15", "label": "RFP deadline"}]
       },
       "sources": ["url1", "url2"],
       "summary": "Brief description of what changed"
     }
   ]
   ```
   If no changes for a state, return: `{"state": "StateName", "changes": {}, "summary": "No changes found"}`

### Status field values (include in each agent's prompt):
- `"Yes"` - RFA/RFP is live and accepting submissions
- `"Active (Rolling)"` - rolling/ongoing application process
- `"Reopened"` - previously closed, now accepting again
- `"Forthcoming"` - officially announced as coming soon
- `"Closed"` - was open, now closed
- `"No - RFA/RFP Planned"` - state confirmed competitive process being developed
- `"No - Competitive Process Planned"` - competitive bidding/grants confirmed, no RFA yet
- `"No - Interest Form Available"` - interest form available but not a full application
- `"No - Notifications Expected"` - state said notifications coming by a specific date
- `"No"` - no downstream opportunity info available

## Step 3: Collect Results and Apply Changes

After all 5 agents complete:

1. Parse the structured results from each agent
2. For each state with changes:
   - Update the relevant fields in `rhtp_results.json`
   - Set `lastUpdated` to today's date (ISO format)
   - Update `nextSteps` with any new dated items, preserving existing future items
   - Do NOT change fields that haven't changed
3. Update the footer date in `index.html` (line ~593) to today's date

## Step 4: Present Changes for Review

Show a summary diff table:

```
| State | Field Changed | Old Value | New Value |
|-------|--------------|-----------|-----------|
```

Also report:
- Total states researched: 50
- Total states with changes: X
- Any states whose program pages were unreachable
- Any states where bot-blocking prevented research

**STOP and wait for user approval before proceeding.**

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

## Usage

```
/update-state-info
```

No arguments needed. The command will research all 50 states by default. If the user adds context like "just hot states" or "just check Ohio", adjust the scope accordingly but follow the same procedure (backup, research, review, push).
