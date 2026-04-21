---
name: startup-job-hunter
description: Multi-platform startup job scraper: scans LinkedIn, startups.gallery, and YC Jobs for target roles. Captures job details, company info, founding team LinkedIn profiles, and resume-JD fit scores. Exports to Google Sheets with incremental updates.
---

# Startup Job Hunter

Multi-platform startup job scanner that searches LinkedIn, startups.gallery, and YC Jobs for target roles. Captures job details, founding team contacts, and exports everything to a Google Sheets tracker with incremental dedup.

---

## TRIGGER

Use this skill when the user wants to:
- Find startup jobs across multiple platforms
- Scan LinkedIn + YC + startups.gallery for open roles
- Build a job tracker with founding team contacts
- Run a recurring startup job search

---

## PHASE 0: COLLECT USER INPUTS

### Required Inputs
1. **Target roles** -- Ask the user for specific role titles they want to search for (e.g., "Growth Lead", "Product Marketing Manager", "GTM Strategist"). Store as a list.
2. **Resume(s)** -- User may upload one or more resume files. Note which resume maps to which role if they specify. Store file paths.
3. **Google Sheet ID** (optional) -- If user has an existing tracker sheet from a previous run, ask for the ID. Otherwise, create a new one.
4. **Location preference** (optional) -- e.g., "Remote", "San Francisco", "New York". Default: no location filter.

```javascript
// Store user inputs
var targetRoles = [] // e.g., ["Growth Lead", "Product Marketing", "GTM"]
var resumePaths = [] // e.g., ["uploads/resume-gtm.pdf", "uploads/resume-pm.pdf"]
var sheetId = null   // existing sheet or null for new
var locationPref = "" // optional
```


---

## PHASE 1: GOOGLE SHEET SETUP

### If new sheet (no existing sheetId):
Create a Google Sheet with title: `"Startup Job Tracker - {date}"`

### Sheet structure -- Single sheet called "Job Tracker"

**Header row (Row 1):**

| Column | Header |
|--------|--------|
| A | Scan Date |
| B | Source |
| C | Company |
| D | Role |
| E | Job Link |
| F | Job Description (summary) |
| G | Location |
| H | Salary Range |
| I | Company LinkedIn |
| J | Company Size |
| K | Funding Stage |
| L | Industry |
| M | Founder 1 Name |
| N | Founder 1 Title |
| O | Founder 1 LinkedIn |
| P | Founder 2 Name |
| Q | Founder 2 Title |
| R | Founder 2 LinkedIn |
| S | Founder 3 Name |
| T | Founder 3 Title |
| U | Founder 3 LinkedIn |
| V | Fit Score (0-100) |
| W | Fit Breakdown |
| X | Applied? |
| Y | Notes |

### If existing sheet:
1. Read all existing rows to build a **dedup set**
2. Dedup key = `lowercase(Company) + "|" + lowercase(Role)`
3. Store in a Set for O(1) lookup

```javascript
// Build dedup set from existing data
var existingKeys = new Set()
var existingData = await google.sheets.getValues({ spreadsheetId: sheetId, range: "Job Tracker!C:D" })
for (var row of existingData) {
  if (row.length >= 2 && row[0] && row[1]) {
    existingKeys.add(row[0].toLowerCase() + "|" + row[1].toLowerCase())
  }
}
```

### New rows go to the TOP (after header row)
When appending new results, insert them starting at Row 2, pushing old data down. This keeps the most recent scan on top.

**HOW TO INSERT AT TOP:** Read existing data (rows 2+), prepend new rows, then write everything back starting at row 2.

---

## PHASE 2: LINKEDIN JOBS (Past 24 Hours)

### Step 0: Expand the user's target role into a FULL keyword set

The user may say a short label like "GTM" or "Growth". That does NOT mean you only search LinkedIn for that exact string. You must expand it into ALL related job titles that fall under that category.

**Keyword expansion table (reference examples):**

| User says | Search with ALL of these keywords |
|-----------|---|
| GTM | GTM, Go-to-Market, Growth Marketing, Growth Manager, Growth Lead, Demand Generation, Revenue Marketing, Marketing Manager SaaS |
| Growth | Growth, Growth Marketing, Growth Manager, Growth Lead, Growth Hacker, User Acquisition, Lifecycle Marketing |
| Marketing | Marketing Manager, Content Marketing, Product Marketing, Brand Marketing, Digital Marketing, Marketing Lead |
| Product | Product Manager, Product Lead, Product Owner, Product Analyst, Technical Product Manager |
| Sales | Account Executive, Sales Development, SDR, BDR, Sales Manager, Revenue, Business Development |
| Operations | Operations Manager, Business Operations, BizOps, RevOps, Sales Operations, Strategy and Operations |
| Engineer | Software Engineer, Backend Engineer, Frontend Engineer, Full Stack, SWE, Developer |
| Design | Product Designer, UX Designer, UI/UX, Design Lead, Visual Designer |

**Rules for keyword expansion:**
- If the user's role is not in the table above, ask the user: "What are all the job titles you'd consider applying to?" and use those as keywords.
- Run a SEPARATE LinkedIn search for each keyword variant.
- After collecting results from all keyword variants, deduplicate by job ID.
- Store which keyword found each job for reference.

### For each keyword in the expanded set:

1. **Navigate to LinkedIn Jobs search:**
```
https://www.linkedin.com/jobs/search/?keywords={encoded_keyword}&f_TPR=r86400
```
- `f_TPR=r86400` = past 24 hours filter
- Add `&location={encoded_location}` if location preference exists

2. **Wait for page to load** (3-5 seconds), then take snapshot

3. **PAGINATE THROUGH ALL RESULTS (up to 10 pages)**
   - After extracting all jobs on the current page, scroll to the bottom
   - Click the next page number or append `&start={n}` to the URL
   - Wait 3 seconds after each page load
   - Stop when: page 10, no more results, all jobs are duplicates, or CAPTCHA appears

4. **Extract job listings** from each page's snapshot:
   - Role title, Company name, Location, Job link, Job ID

5. **Relevance filter AFTER collection:**
   - Filter out clearly irrelevant roles (e.g., engineering roles when user wants marketing)
   - When in doubt, KEEP the job -- the Fit Score will handle fine-grained filtering

6. **Click into each RELEVANT job** for full details:
   - Extract 2-3 sentence JD summary, salary range, company LinkedIn URL

7. **Check dedup** -- skip if company|role already exists

8. **For each NEW job -- check company size on LinkedIn:**
   - Navigate to company LinkedIn page, record employee count

9. **If company has <200 employees -- scrape founding team:**
   - Navigate to company People page
   - Search/filter for "founder", "CEO", "CTO"
   - Click into each founder's profile to get name, title, LinkedIn URL
   - Record up to 3 founders

10. **Store results** in the collector array

### LinkedIn rate-limit awareness:
- Wait 2-3 seconds between page loads
- Process max 15-20 job listings per role to avoid throttling


---

## PHASE 3: STARTUPS.GALLERY (Past 1 Week)

### Step 0: Use the SAME expanded keyword set from Phase 2

### For each keyword in the expanded set:

1. **Navigate to** `https://startups.gallery/jobs`

2. **Enter keyword in search box:**
   - Find the Role textbox (placeholder: "e.g. Designer")
   - Clear any previous search text
   - Type the current keyword and press Enter or wait for results to filter
   - Wait 3 seconds for results to load

3. **Scan ALL job listings (not just the first page):**
   - Each listing shows: Role title (h2), Company name, Location, Posted date
   - Posted date format: "Posted on Apr 17, 2026"
   - **Only collect jobs posted within the past 7 days**

4. **For each qualifying job (within 7 days):**
   - Record: Role title, Company name, Location, Posted date, External apply link
   - Check dedup -- skip if already exists

5. **Scroll down and click "Load More" REPEATEDLY** to load all recent jobs

6. **After all keywords are searched, deduplicate** by company + role across all keyword searches

7. **For each NEW job -- visit company LinkedIn:**
   - Find company LinkedIn page
   - Extract company size, industry, funding stage
   - If <200 employees, scrape founders (same process as Phase 2)
   - Record up to 3 founders with name, title, LinkedIn URL

8. **Store results** with source = "startups.gallery"

---

## PHASE 4: YC JOBS (Y Combinator)

### Role category mapping:
YC Jobs uses preset role categories. Map user's target roles to the closest category:

| User Role Keywords | YC Category | URL Path |
|-------------------|-------------|----------|
| Engineer, Developer, SWE | Software Engineer | /jobs (default) |
| Designer, UI, UX | Design and UI/UX | /jobs/role/designer |
| Product Manager, PM | Product | /jobs/role/product-manager |
| HR, Recruiter, Talent | Recruiting and HR | /jobs/role/recruiting-hr |
| Sales, GTM, Growth, BDR | Sales | /jobs/role/sales-manager |
| Research, Scientist, ML | Science | /jobs/role/science |

### For each mapped category:

1. **Navigate to** `https://www.ycombinator.com/jobs/role/{category}`

2. **Scroll through ALL listings:**
   - Collect jobs posted within the past 7 days (parse relative time like "3 days ago")
   - Scroll and lazy-load until no new jobs or all are older than 7 days

3. **For each qualifying job:**
   - Click into job detail page
   - Extract: Full job title, Company name, YC batch, Location, Salary, JD summary
   - Record the direct job page URL as the apply link

4. **Check dedup** -- skip if already exists

5. **For each NEW job -- visit YC company page AND get LinkedIn founder profiles:**
   - Visit YC company page for description, batch, industry tags, founder names
   - If YC page shows founder names but no LinkedIn links:
     - Google search: `site:linkedin.com/in "{founder name}" "{company name}"`
     - Click into LinkedIn profile to verify and record URL
   - Fallback: Visit LinkedIn company page + People page for founder research
   - Always record company LinkedIn URL

6. **Store results** with source = "YC Jobs"


---

## PHASE 5: WRITE TO GOOGLE SHEET

### Compile all new jobs

Format each job as a row matching the 25-column header structure (A through Y).

### Insert at top (Row 2):

```javascript
// Read existing data (excluding header)
var existing = await google.sheets.getValues({ spreadsheetId: sheetId, range: "Job Tracker!A2:Y" })

// Prepend new rows
var allData = [...newRows, ...(existing || [])]

// Clear and rewrite
await google.sheets.clearValues({ spreadsheetId: sheetId, range: "Job Tracker!A2:Y" })
await google.sheets.updateValues({ 
  spreadsheetId: sheetId, 
  range: "Job Tracker!A2", 
  values: allData 
})
```

---

## PHASE 5.5: RESUME-JD FIT EVALUATION

For every new job collected, score how well the user's resume matches the JD.

### Pre-requisite: Parse User's Resume(s)

Read and parse each uploaded resume using the `read-files` skill. Extract skills, industries, experience years, seniority level, functional areas, tools, achievements, and education.

### Scoring Framework (0-100 Scale)

| Dimension | Weight | What to Compare |
|-----------|--------|----------------|
| **Role Relevance** | 25% | Does the JD match the user's functional area? |
| **Industry/Market Fit** | 20% | B2B vs B2C alignment, vertical overlap |
| **Skills Match** | 20% | Required skills overlap with resume |
| **Seniority Alignment** | 15% | Experience level match |
| **Tools and Platform Match** | 10% | Specific tools overlap (Salesforce, HubSpot, etc.) |
| **Domain Depth** | 10% | Demonstrable depth in JD's domain |

**Score interpretation:**
- 80-100: Strong Fit -- Apply immediately
- 60-79: Moderate Fit -- Apply with tailored resume
- 40-59: Stretch -- Only if mission is compelling
- 0-39: Weak Fit -- Skip unless referral exists

**Breakdown string format:**
`"Role:{n}/25 | Industry:{n}/20 | Skills:{n}/20 | Seniority:{n}/15 | Tools:{n}/10 | Domain:{n}/10 -- {one-sentence qualitative note}"`

### Evaluation Rules
- Be honest and strict -- do not inflate scores
- B2B vs B2C is a critical signal
- "Nice to have" vs "Must have" matters
- Use the FULL JD context, not just the summary
- Sort final newJobs array by fitScore descending before writing to sheet

---

## PHASE 6: SUMMARY REPORT

After writing to the sheet, report to the user:

```
Startup Job Scan Complete -- {date}

Results Summary:
- LinkedIn: {n} new jobs found (past 24h)
- startups.gallery: {n} new jobs found (past 7 days)
- YC Jobs: {n} new jobs found (past 7 days)
- Total new: {n} | Duplicates skipped: {n} | Irrelevant filtered: {n}

Fit Score Distribution:
- Strong (80+): {n} jobs
- Moderate (60-79): {n} jobs
- Stretch (40-59): {n} jobs
- Weak (<40): {n} jobs
- Top 3 highest-fit jobs:
  1. {company} -- {role} -- {score}/100
  2. {company} -- {role} -- {score}/100
  3. {company} -- {role} -- {score}/100

Google Sheet: {sheet_url}
Keywords searched: {full_keyword_list}
Location filter: {location or "None"}

Next run tip: Re-run this skill anytime. It will only add NEW jobs not already in the sheet.
```

---

## IMPORTANT BEHAVIORAL RULES

### Rate Limiting and Safety
- Wait 2-3 seconds between page navigations on LinkedIn
- Wait 1-2 seconds between page loads on other sites
- Max 20 jobs per source per role to avoid session timeouts
- If any site shows CAPTCHA or login wall, note it and move on

### Data Quality
- JD Summary should be 2-3 sentences max
- Only collect people with clear leadership titles (Founder, Co-Founder, CEO, CTO, COO)
- Record company size as shown on LinkedIn (e.g., "11-50", "51-200")
- If company size is 200+, skip founder scraping

### Founder Data Completeness (NON-NEGOTIABLE)
- Every job must have founding team researched (unless company is 200+ employees)
- Founder LinkedIn URL is the most important field
- Required data per founder: Full name, Current title, LinkedIn profile URL
- Do not batch-skip founder research to save time

### Dedup Logic
- Key: `lowercase(company_name) + "|" + lowercase(role_title)`
- Normalize company names: remove "Inc.", "Ltd.", "Co." suffixes
- Check BEFORE visiting company LinkedIn (save time)

### Error Handling
- If a page fails to load, wait 5 seconds and retry once
- If retry fails, log the error and move to next job
- If LinkedIn is not logged in, results will be limited -- warn the user

### Incremental Updates
- Always read the existing sheet first to build dedup set
- New jobs go to the TOP of the sheet (most recent scan first)
- Users can add notes in "Applied?" and "Notes" columns -- these must be preserved
