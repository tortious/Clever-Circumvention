## ANALYSIS

Reviewing your context, I see these high-leverage automation opportunities:

**Critical gaps:**
- Manual case intake creates bottlenecks and data inconsistencies
- Calendar events disconnected from case data
- Document metadata relies on manual tagging
- No automated deadline tracking/alerts
- Mark can't self-serve basic case info
- Email parsing exists but isn't fully integrated into workflows
- Financial tracking is manual and error-prone

**Structural advantages you already have:**
- Hub-spoke architecture ready for centralized automation
- Managed metadata foundation (CourtNumber) for search/matching
- Azure Function App infrastructure working
- Graph API patterns established
- Clear separation between structured data (hub lists) and documents (team sites)

**Implementation priorities:**
1. Automate what currently blocks you (case intake, metadata extraction)
2. Connect siloed systems (calendar ↔ cases, email ↔ lists)
3. Surface insights you can't see now (cross-case patterns, deadline exposure)
4. Reduce Mark's dependency on you for routine info

---

## 20 CONCRETE SKILLS/AGENTS

### **1. NEW MATTER SITE GENERATOR**

**Problem:** Creating a new case site manually takes 20+ minutes and is error-prone (wrong permissions, missing folders, incomplete metadata).

**Trigger:** HTTP request from SharePoint list button OR Power Automate flow when new item added to CLARK master list

**Inputs:**
- Case number (from list item)
- Client name
- Claim number
- Case type (PI, premises, MVA)
- Assigned attorney (Mark)
- Defense counsel firm name
- Policy limits

**Outputs:**
- New Team site created from template
- Site URL written back to CLARK list item
- Standard document libraries created (Pleadings, Discovery, Correspondence, Medical, Expert)
- Metadata columns stamped on libraries
- Permission group applied (Mark + you as owners, paralegal as member)
- Teams team created with case number as name
- Initial folders created in each library
- Welcome page deployed with case summary

**Integration Points:**
- **Graph API:** `POST /sites/{hub-site-id}/sites` (create site)
- **Graph API:** `POST /groups` (create Microsoft 365 Group for Team)
- **SharePoint REST:** `/_api/web/lists/getbytitle('Documents')/items` (add folders)
- **Graph API:** `PATCH /sites/{site-id}/lists/{list-id}/items/{item-id}` (update CLARK list with site URL)
- **Runs in:** Azure Function App (HTTP-triggered)

**MVP Implementation:**
1. Register app in Entra with `Sites.FullControl.All`, `Group.ReadWrite.All`
2. Create Azure Function with cert-based auth
3. Define site template JSON (libraries, columns, navigation)
4. Build Graph API chain:
   - Create Group → Get site ID → Add libraries → Set permissions → Update hub list
5. Add Power Automate button in CLARK list view that calls Function with item ID
6. Test with 3 different case types

**Key Risks + Mitigations:**
- **Risk:** Site creation timeout (Graph can take 30+ seconds)
  - *Mitigation:* Return 202 Accepted immediately, poll for completion, update list async
- **Risk:** Duplicate sites if button clicked twice
  - *Mitigation:* Check CLARK list for existing site URL before creating, lock item during creation
- **Risk:** Permission inheritance breaks after site creation
  - *Mitigation:* Explicitly break inheritance and set groups via Graph, verify in post-creation check
- **Risk:** Template drift (libraries change but template doesn't)
  - *Mitigation:* Store template JSON in SharePoint list, version it, document in CLAUDE.md

**Operator UX:**
1. Navigate to CLARK list
2. Find new case (added manually or via email parser)
3. Click "Provision Site" button in item menu
4. Wait 10 seconds, refresh → site URL appears in "CaseSiteURL" column
5. Click link to verify setup

---

### **2. CASE METADATA EXTRACTOR (Email Parser v2)**

**Problem:** Assignment emails from First Chicago contain all case data, but manual entry into CLARK list takes 5-10 minutes and introduces typos.

**Trigger:** Email arrives in shared mailbox with subject containing "Assignment" OR "New Claim"

**Inputs:**
- Email body (HTML or plain text)
- PDF attachment (assignment letter)
- Sender address (to verify it's First Chicago)

**Outputs:**
- New item in CLARK list with extracted fields populated:
  - Court number
  - Claim number
  - Plaintiff name
  - Date of loss
  - Policy limits
  - Adjuster name + email
  - Incident type
  - Venue
- PDF saved to temp library with extracted metadata stamped
- Email archived to "Processed Assignments" folder
- Teams notification to you with extraction confidence score

**Integration Points:**
- **Graph API:** `GET /me/mailFolders/{folder-id}/messages?$filter=...` (monitor inbox)
- **Azure AI Document Intelligence:** Extract structured data from PDF
- **Graph API:** `POST /sites/{hub-site-id}/lists/{CLARK-list-id}/items` (create list item)
- **Graph API:** `POST /teams/{team-id}/channels/{channel-id}/messages` (notify)
- **Runs in:** Azure Function (timer-triggered every 5 minutes OR Logic App with email trigger)

**MVP Implementation:**
1. Set up shared mailbox delegation in Entra
2. Register app with `Mail.ReadWrite`, `Sites.ReadWrite.All`, `ChannelMessage.Send`
3. Create Azure AI Document Intelligence resource (F0 tier free)
4. Build extraction logic:
   - Regex patterns for claim #, court #, policy limits
   - Named entity recognition for names/dates
   - Fallback to LLM (GPT-4o-mini) for complex formats
5. Map extracted fields to CLARK list columns (with validation)
6. Add confidence scoring (high/medium/low based on field completeness)
7. Human-in-loop: low confidence → Teams notification for review before list creation

**Key Risks + Mitigations:**
- **Risk:** Email format changes break extraction
  - *Mitigation:* Log failed extractions to separate list, alert weekly, maintain regex library
- **Risk:** False positives (non-assignment emails processed)
  - *Mitigation:* Strict sender whitelist + subject keyword combo
- **Risk:** Duplicate case creation
  - *Mitigation:* Check CLARK list for existing claim # before creating, update existing if found
- **Risk:** Incorrect policy limit extraction (typo could be costly)
  - *Mitigation:* Require human approval for policy limits >$100k, flag anomalies

**Operator UX:**
1. Email arrives in shared inbox (no action needed)
2. Receive Teams notification: "New case extracted: Claim #12345 (confidence: HIGH)"
3. Click link to review CLARK list item
4. Verify data, click "Approve & Provision Site" button
5. If low confidence, manually correct fields before approving

---

### **3. CALENDAR-TO-CASE DASHBOARD**

**Problem:** Mark has hearings/depositions in Outlook, but you can't quickly see which case they belong to or access the case files from his calendar.

**Trigger:** Manual page load OR scheduled refresh every 30 minutes

**Inputs:**
- Mark's Outlook calendar (next 30 days)
- CLARK master list (case numbers, site URLs, client names)

**Outputs:**
- HTML dashboard on SharePoint page showing:
  - Today's events as cards
  - This week's events grouped by day
  - Each card shows: event title, case number (matched), client name, time, clickable link to case site
- Unmatched events highlighted in yellow for manual review
- "Add to case" button on unmatched events

**Integration Points:**
- **Graph API:** `GET /users/{mark-upn}/calendar/calendarView?startDateTime=...&endDateTime=...`
- **SharePoint REST:** `/_api/web/lists/getbytitle('CLARK')/items?$select=...`
- **Runs in:** SharePoint page with embedded script OR SPFx web part OR Azure Function returning JSON consumed by list formatting

**MVP Implementation (low-friction approach):**
1. Create Power Automate flow (scheduled, every 30 min):
   - Get Mark's calendar events (next 7 days)
   - Get all active cases from CLARK list
   - Match events to cases using fuzzy logic:
     - Extract court # from event subject (regex)
     - Search CLARK list for matching court #
     - Fallback: match on client name substring
   - Write results to "CalendarMatches" list with columns: EventID, EventTitle, CaseID, MatchConfidence, CaseSiteURL
2. Create SharePoint page with list view web part showing CalendarMatches
3. Apply JSON formatting to render as cards:
   - Green border if MatchConfidence = High
   - Yellow if Medium (needs review)
   - Red if No match
   - Click card → navigate to case site
4. Add quick action buttons: "Confirm Match", "Link to Different Case", "Ignore"

**Alternative (if you want real-time):**
- Azure Function endpoint that returns JSON
- Embed via `<iframe>` or Power Apps component
- Caches calendar data for 15 min to avoid throttling

**Key Risks + Mitigations:**
- **Risk:** Calendar subject lines inconsistent (missing court #)
  - *Mitigation:* Train Mark to use format "Hearing - Smith v. Jones - 2024-L-1234", provide Outlook template
- **Risk:** Multiple cases match (same client, different matters)
  - *Mitigation:* Show all matches, let operator pick correct one
- **Risk:** Graph throttling if checking every 5 min
  - *Mitigation:* Use delta queries, cache for 30 min, only refresh on user action
- **Risk:** Permissions (you can't read Mark's calendar by default)
  - *Mitigation:* Mark delegates calendar to you OR app uses app-only permission `Calendars.Read`

**Operator UX:**
1. Open "Daily Docket" SharePoint page (bookmark it)
2. See today's events as cards at top
3. Click card → opens case site in new tab
4. Yellow cards → review, click "Link to Case" dropdown, select correct case
5. Refresh page manually if you add new calendar event

---

### **4. DEADLINE TRACKER SYNC**

**Problem:** Court deadlines are scattered across Outlook, emails, and case files. No centralized view of upcoming deadlines with automatic alerts.

**Trigger:** 
- Calendar event created/updated in Mark's Outlook
- Email received with keyword "hearing", "trial", "discovery deadline"
- Manual "Add Deadline" button in case site

**Inputs:**
- Calendar event details (title, date, location)
- Associated case number
- Deadline type (hearing, discovery response, filing deadline)

**Outputs:**
- New item in "Deadlines" list (hub level) with columns:
  - CaseNumber (managed metadata)
  - DeadlineDate
  - DeadlineType
  - Description
  - DaysUntil (calculated)
  - Status (Upcoming, Completed, Missed)
- Email alert 7 days before deadline
- Teams notification 1 day before
- Dashboard view showing all deadlines in next 30 days

**Integration Points:**
- **Graph API:** `GET /users/{mark-upn}/calendar/events` with delta query
- **Graph API:** `POST /sites/{hub-site-id}/lists/Deadlines/items`
- **Power Automate:** Send alerts via Outlook/Teams
- **Runs in:** Combination of Graph subscription webhook + Azure Function + Power Automate

**MVP Implementation:**
1. Create "Deadlines" list at hub with required columns
2. Set up Graph webhook subscription for Mark's calendar (notifies Function on event changes)
3. Azure Function processes webhook:
   - Parse event for case number (subject line regex)
   - Determine deadline type (keyword matching: "hearing" → Hearing, "disc" → Discovery)
   - Create/update Deadlines list item
4. Power Automate flow (scheduled daily 8am):
   - Get all Deadlines where DaysUntil = 7
   - Send summary email to Mark + you
5. Second flow for DaysUntil = 1 (Teams alert)
6. Create dashboard page with list view filtered to next 30 days, sorted by date

**Key Risks + Mitigations:**
- **Risk:** Webhook expires after 3 days (Graph limitation)
  - *Mitigation:* Auto-renew subscription via scheduled Function, monitor expiry in App Insights
- **Risk:** Missed deadline not flagged
  - *Mitigation:* Daily reconciliation job checks for Status = Upcoming where DeadlineDate < Today, marks as Missed, sends urgent alert
- **Risk:** Duplicate deadlines if event updated multiple times
  - *Mitigation:* Use event ID as unique key, upsert logic instead of create
- **Risk:** Wrong case number parsed
  - *Mitigation:* Flag for review if case # not found in CLARK list, send Teams notification

**Operator UX:**
1. Mark creates calendar event "Motion Hearing - Smith v. Jones - 2024-L-1234 - 3/15/25 9am"
2. Within 5 min, deadline appears in Deadlines list (automated)
3. You review dashboard daily, see all upcoming deadlines
4. 7 days before, you both get email summary
5. 1 day before, Teams notification pings
6. After deadline passes, mark as "Completed" in list

---

### **5. INCOMING DOC TAGGER**

**Problem:** PDFs and Word docs arrive via email or are uploaded to case sites without metadata. Finding them later requires full-text search or manual browsing.

**Trigger:** 
- Document uploaded to any case site document library
- Email attachment saved to SharePoint

**Inputs:**
- Document file (PDF, DOCX, MSG)
- Source location (which library, which case site)

**Outputs:**
- Metadata stamped on document:
  - DocumentType (Motion, Discovery, Correspondence, Medical, Expert Report)
  - PartyName (extracted from content)
  - DateReceived
  - CourtNumber (linked to managed metadata)
- Document moved to correct library if uploaded to wrong one
- OCR applied if scanned PDF
- Searchable text indexed

**Integration Points:**
- **Graph API:** `GET /sites/{site-id}/drive/items/{item-id}/content` (download doc)
- **Azure AI Document Intelligence:** OCR + classification
- **SharePoint REST:** `/_api/web/lists/getbytitle('Documents')/items({item-id})` (update metadata)
- **Runs in:** Azure Function triggered by SharePoint webhook (library change)

**MVP Implementation:**
1. Register Graph change subscription for all case site document libraries (one subscription per site, or use shared webhook)
2. Azure Function receives webhook notification when doc uploaded:
   - Download document
   - Extract text (Azure AI or Apache Tika)
   - Classify document type using keyword matching:
     - "MOTION TO" → Motion
     - "INTERROGATORIES" → Discovery Request
     - "MEDICAL RECORDS" → Medical
   - Extract entities (court #, party names) via regex + NER
   - Update SharePoint item metadata via REST API
3. If scanned PDF (no text layer), run OCR first, save OCRed version
4. Log all tagging actions to audit list

**Key Risks + Mitigations:**
- **Risk:** Classification accuracy varies (85-95%)
  - *Mitigation:* Set confidence threshold, flag low-confidence docs for manual review, improve model over time
- **Risk:** Webhook overwhelm if bulk upload (100+ docs)
  - *Mitigation:* Queue messages in Azure Service Bus, process async with retry logic
- **Risk:** Large PDFs timeout Function (300-page depo transcript)
  - *Mitigation:* Durable Functions for long-running tasks, async processing
- **Risk:** Incorrect metadata overwrites correct manual entry
  - *Mitigation:* Never overwrite user-entered metadata, only populate empty fields

**Operator UX:**
1. Receive email with PDF attachment
2. Forward to case email address OR drag into SharePoint library
3. Wait 30 seconds, refresh library view
4. See metadata auto-populated: DocumentType = "Motion", CourtNumber = "2024-L-1234"
5. Verify accuracy, manually correct if needed
6. Low-confidence docs appear in "Review Queue" view

---

### **6. ANSWER DUE DATE CALCULATOR**

**Problem:** Computing answer deadlines manually is error-prone (Illinois rules: 28 days from service, +3 for mail service, skip holidays/weekends).

**Trigger:** 
- New complaint uploaded to case site
- Manual button in CLARK list "Calculate Answer Due"
- Service date entered in case metadata

**Inputs:**
- Service date
- Service method (personal, mail, email)
- Jurisdiction (defaults to Illinois)

**Outputs:**
- Answer due date calculated per local rules
- Deadline added to Deadlines list
- Calendar event created in Mark's Outlook
- Reminder set for 7 days before due date

**Integration Points:**
- **SharePoint REST:** Get service date from case metadata
- **Business logic:** Date calculation with holiday calendar
- **Graph API:** `POST /users/{mark-upn}/calendar/events` (create calendar event)
- **Runs in:** Power Automate flow OR Azure Function

**MVP Implementation:**
1. Create "Answer Due Date Calculator" Power Automate flow (manual trigger from CLARK list)
2. Get service date + method from list item
3. Apply rules:
   - Personal service: +28 days
   - Mail service: +31 days (28 + 3)
   - Email service: +28 days (verify local rule)
4. Check against holiday calendar (use Graph API for Outlook holidays OR maintain custom list)
5. If deadline falls on weekend/holiday, move to next business day
6. Create calendar event with title "Answer Due - [Case Name] - [Court #]"
7. Add to Deadlines list
8. Send confirmation email with calculated date

**Key Risks + Mitigations:**
- **Risk:** Wrong rule applied (jurisdiction varies)
  - *Mitigation:* Validate jurisdiction field in CLARK list, warn if not Illinois
- **Risk:** Holiday calendar outdated
  - *Mitigation:* Sync with Illinois court calendar annually, manual override option
- **Risk:** Service date typo
  - *Mitigation:* Validate date format, warn if >60 days in past or future
- **Risk:** Calculated date changes if service date corrected
  - *Mitigation:* Detect change, recalculate, update calendar + deadlines, notify

**Operator UX:**
1. Open new case in CLARK list
2. Enter service date: "2025-02-15"
3. Select service method: "Mail"
4. Click "Calculate Answer Due" button
5. See message: "Answer due: March 18, 2025 (31 days from service)"
6. Verify calendar event created, deadline in Deadlines list
7. If wrong, manually adjust service date and recalculate

---

### **7. PLEADING TEMPLATE FILLER**

**Problem:** Generating answers, motions, and notices requires copying boilerplate and manually filling 15+ fields from case data.

**Trigger:** 
- Manual command in case site: "Generate Answer"
- Button in CLARK list for common pleadings

**Inputs:**
- Pleading type (Answer, Motion to Dismiss, Notice of Motion)
- Case number (from CLARK list)
- Case data: parties, attorneys, court, judge

**Outputs:**
- Word document in Pleadings library with:
  - All boilerplate populated
  - Caption filled
  - Case-specific data inserted
  - Signature block for Mark
- Document metadata stamped (DocumentType, CaseNumber)
- Ready for review and e-filing

**Integration Points:**
- **SharePoint REST:** Get case data from CLARK list
- **Word template:** .docx with content controls
- **Graph API:** `POST /sites/{site-id}/drive/items/{folder-id}/children` (upload doc)
- **Runs in:** Azure Function OR Power Automate with "Populate Word template" action

**MVP Implementation (Power Automate - easier):**
1. Create Word templates in SharePoint library with content controls:
   - `{CourtName}`
   - `{CaseNumber}`
   - `{PlaintiffName}`
   - `{DefendantName}`
   - etc.
2. Build Power Automate flow (manual trigger from CLARK list):
   - Get case item from list
   - Use "Populate a Word template" action with template ID
   - Map list fields to content controls
   - Save populated doc to case site Pleadings library
   - Notify via Teams
3. Add button in CLARK list view with flow trigger

**Alternative (Azure Function - more flexible):**
1. Use Open XML SDK to manipulate Word docs
2. Replace placeholders via code
3. Supports complex logic (conditional sections, tables, etc.)

**Key Risks + Mitigations:**
- **Risk:** Template changes require flow updates
  - *Mitigation:* Use dynamic content controls, version templates, document in CLAUDE.md
- **Risk:** Missing data breaks document
  - *Mitigation:* Validate required fields before generation, show error if incomplete
- **Risk:** Output formatting broken (indents, spacing)
  - *Mitigation:* Test templates thoroughly, use styles instead of direct formatting
- **Risk:** Wrong case data merged
  - *Mitigation:* Show preview before final save, require confirmation

**Operator UX:**
1. Open case in CLARK list
2. Click "Generate Answer" button
3. Select template: "Answer - General Denial"
4. Preview populated document
5. Click "Save to Case Site"
6. Document appears in Pleadings library within 10 seconds
7. Download, review, e-file

---

### **8. DISCOVERY RESPONSE PACKAGER**

**Problem:** Responding to discovery requests requires collecting documents from multiple libraries, organizing by request number, creating a cover letter, and packaging for production.

**Trigger:** Manual command "Package Discovery Response"

**Inputs:**
- Request numbers to respond to (e.g., "Requests 1-5, 10-15")
- Documents to include (selected from libraries)
- Case number

**Outputs:**
- Folder structure in Discovery library:
  - Response_to_Requests_1-5/
    - Request_1/ (docs)
    - Request_2/ (docs)
    - Cover_Letter.docx
    - Index.xlsx (Bates-stamped log)
- Cover letter generated with template
- Index spreadsheet listing all documents + Bates numbers
- ZIP file for email transmission
- Service certificate generated

**Integration Points:**
- **SharePoint REST:** Copy documents to response folder
- **Graph API:** Create folder structure
- **Excel automation:** Generate index spreadsheet
- **Runs in:** Azure Function OR Power Automate

**MVP Implementation:**
1. Create SharePoint form to capture request numbers + select documents
2. Power Automate flow:
   - Create folder structure in Discovery library
   - Copy selected documents to appropriate request folders
   - Generate Bates numbers (sequential: CASEID-0001, CASEID-0002...)
   - Stamp Bates on PDF footers (Azure Function with PDF library)
   - Create Excel index with columns: BatesNumber, FileName, RequestNumber, Description
   - Populate Word cover letter template
   - ZIP entire folder (using custom action or Function)
   - Upload ZIP to "Ready for Service" folder
   - Notify via Teams
3. Add manual review step before finalizing

**Key Risks + Mitigations:**
- **Risk:** Bates numbering collision (multiple responses in progress)
  - *Mitigation:* Lock Bates counter during generation, use atomic increment
- **Risk:** Wrong documents included
  - *Mitigation:* Require manual selection + review step
- **Risk:** Large ZIP fails to create (>2GB)
  - *Mitigation:* Split into multiple ZIPs if over threshold, warn user
- **Risk:** Confidential docs accidentally included
  - *Mitigation:* Flag privileged docs in metadata, exclude automatically, manual override required

**Operator UX:**
1. Click "Package Discovery Response" button in case site
2. Select request numbers from dropdown: "1-5"
3. Browse libraries, select documents to include
4. Review selected documents (15 files)
5. Click "Generate Package"
6. Wait 2 minutes (processing)
7. Receive Teams notification: "Discovery response ready"
8. Open "Ready for Service" folder, download ZIP
9. Verify cover letter + index, e-serve

---

### **9. SERVICE EMAIL PROCESSOR**

**Problem:** Emails confirming service, filing, or opposing counsel correspondence arrive but aren't logged to case files. Manual forwarding/saving is inconsistent.

**Trigger:** Email arrives in shared mailbox with keywords "served", "filed", "e-service"

**Inputs:**
- Email subject + body
- Sender address
- Attachments
- Timestamp

**Outputs:**
- Email saved to case site Correspondence library
- Metadata tagged: EmailType (Service, Filing, General), Sender, DateReceived
- Case status updated if service confirmed (e.g., "Answer Filed" → update status in CLARK list)
- Notification to Mark if critical (e.g., motion served requiring response)

**Integration Points:**
- **Graph API:** `GET /me/mailFolders/{inbox}/messages?$filter=...`
- **SharePoint REST:** Save email as MSG file, stamp metadata
- **Runs in:** Azure Function (timer-triggered) OR Logic App (email trigger)

**MVP Implementation:**
1. Monitor shared inbox for emails with keywords
2. Azure Function processes each email:
   - Extract case number from subject (regex)
   - Classify email type:
     - "proof of service" → Service
     - "Notice of Filing" → Filing
     - From opposing counsel → General Correspondence
   - Download email as .MSG file
   - Upload to case site Correspondence library
   - Update CLARK list if status change needed (e.g., "Answer Filed" date)
   - If critical (motion served), send Teams alert
3. Move processed email to "Processed" folder
4. Log all actions to audit list

**Key Risks + Mitigations:**
- **Risk:** False positives (irrelevant emails processed)
  - *Mitigation:* Strict keyword combo + sender whitelist
- **Risk:** Case number not detected
  - *Mitigation:* Save to "Unmatched Emails" library for manual review
- **Risk:** Duplicate processing if email fetched twice
  - *Mitigation:* Track processed email IDs in list, skip if already processed
- **Risk:** Missed critical email (spam filter, typo in subject)
  - *Mitigation:* Daily report of unprocessed emails in inbox

**Operator UX:**
1. Email arrives: "Notice of Filing - Smith v. Jones - 2024-L-1234"
2. Automated processing within 5 min (no action needed)
3. Check Teams notification: "Answer filed by opposing counsel"
4. Navigate to case site Correspondence library, see email saved
5. Verify metadata correct
6. If unmatched, review "Unmatched Emails" list weekly, manually assign to cases

---

### **10. SETTLEMENT AUTHORITY TRACKER**

**Problem:** Tracking policy limits, demands, and settlement authority across multiple cases requires spreadsheets. No alerts when demand exceeds limits.

**Trigger:** 
- Policy limit entered in CLARK list
- Demand entered or updated
- Monthly review report generation

**Inputs:**
- Policy limits (per case)
- Current demand amount
- Settlement authority granted by client/carrier
- Case expenses

**Outputs:**
- Dashboard showing:
  - Cases where demand > 80% of policy limits (exposure risk)
  - Cases with settlement authority gaps (demand exceeds authority)
  - Total exposure across all cases
- Alerts when demand updated to exceed policy limits
- Monthly report emailed to Mark

**Integration Points:**
- **SharePoint REST:** Get policy limits + demand from CLARK list
- **Power BI:** Dashboard (optional, or use formatted list views)
- **Power Automate:** Alert emails
- **Runs in:** Scheduled Power Automate flow

**MVP Implementation:**
1. Add columns to CLARK list:
   - PolicyLimits (currency)
   - CurrentDemand (currency)
   - SettlementAuthority (currency)
   - ExposureLevel (calculated: Low, Medium, High)
2. Create calculated column: ExposureLevel
   - If Demand > 80% of PolicyLimits → High
   - If Demand > 50% → Medium
   - Else → Low
3. Power Automate flow (daily 8am):
   - Get all cases with ExposureLevel = High
   - Send summary email to Mark
4. Second flow: when CurrentDemand updated
   - If new demand > PolicyLimits → immediate Teams alert
5. Create SharePoint page with list view filtered to High exposure, sorted by demand desc
6. Monthly report: export to Excel, email

**Key Risks + Mitigations:**
- **Risk:** Policy limits data incomplete/inaccurate
  - *Mitigation:* Require policy limits before case can be marked "Active", validation rule
- **Risk:** Demand not updated timely
  - *Mitigation:* Remind Mark monthly to review demands
- **Risk:** False alerts (demand typo)
  - *Mitigation:* Validate currency format, warn if >10x previous demand
- **Risk:** Authority limits not coordinated with carrier
  - *Mitigation:* Link to carrier contact, reminder to confirm authority quarterly

**Operator UX:**
1. Receive demand letter, update CLARK list CurrentDemand field: $250,000
2. Policy limits: $100,000
3. Immediate Teams alert: "EXPOSURE: Smith v. Jones - Demand ($250k) exceeds policy ($100k)"
4. Review dashboard weekly: see 5 cases flagged High exposure
5. Coordinate with adjuster for authority increase
6. Update SettlementAuthority field once obtained
7. Monthly report auto-sent, review in staff meeting

---

### **11. COURT REPORTER ORDER AGENT**

**Problem:** After depositions/hearings, ordering transcripts requires manually emailing court reporter, tracking order status, and following up.

**Trigger:** 
- Calendar event ends (deposition/hearing)
- Manual "Order Transcript" button

**Inputs:**
- Event details (deponent name, date, court reporter)
- Case number
- Transcript type (expedited, regular)
- Delivery preference (paper, PDF, both)

**Outputs:**
- Email sent to court reporter with order details
- Tracking item created in "Transcript Orders" list
- Reminder set for expected delivery date (+14 days regular, +7 expedited)
- Follow-up email sent if not received by due date
- Invoice logged when transcript arrives

**Integration Points:**
- **Graph API:** Send email, create calendar reminder
- **SharePoint:** "Transcript Orders" list
- **Power Automate:** Reminder + follow-up logic
- **Runs in:** Power Automate flow

**MVP Implementation:**
1. Create "Transcript Orders" list at hub with columns:
   - CaseNumber, DeponentName, DepositionDate, ReporterName, ReporterEmail, OrderDate, ExpectedDeliveryDate, Status (Ordered, Received, Cancelled), TranscriptType
2. Power Automate flow (manual trigger from calendar event OR case site):
   - Get event details
   - Populate email template:
     - "Please prepare transcript of [Deponent] deposition on [Date] for [Case Name]. Delivery: [Type]. Contact: [Email]"
   - Send email to reporter
   - Create item in Transcript Orders list
   - Calculate ExpectedDeliveryDate (OrderDate + 14 or 7 days)
   - Set reminder
3. Second flow (daily check):
   - Get all orders where Status = Ordered AND ExpectedDeliveryDate < Today
   - Send follow-up email to reporter
   - Update Status to "Overdue"
   - Notify Mark
4. Manual update when transcript received: mark Status = Received

**Key Risks + Mitigations:**
- **Risk:** Reporter email bounces
  - *Mitigation:* Validate email before sending, maintain reporter contact list
- **Risk:** Transcript arrives but not logged
  - *Mitigation:* Reminder in Teams to update status when received
- **Risk:** Wrong reporter contacted
  - *Mitigation:* Pre-populate reporter from calendar event location or contact field
- **Risk:** Expedited vs. regular confusion
  - *Mitigation:* Require explicit selection, confirm before sending order

**Operator UX:**
1. Deposition ends, return to office
2. Click "Order Transcript" button in calendar event OR case site
3. Verify pre-filled details: deponent name, date, reporter
4. Select type: "Expedited (7 days)"
5. Click "Send Order"
6. Confirmation message: "Order sent to Jane Smith, Reporter. Expected delivery: [Date]"
7. Receive reminder 7 days later if not received
8. When transcript arrives, mark Status = Received in Transcript Orders list

---

### **12. CROSS-CASE SEARCH AGENT**

**Problem:** Finding similar cases (same injury type, venue, opposing counsel) requires manual memory or searching through all case files.

**Trigger:** 
- Manual search query
- "Find Similar Cases" button in case site

**Inputs:**
- Search criteria: injury type, venue, opposing counsel, adjuster, policy carrier, outcome

**Outputs:**
- List of matching cases with similarity score
- Key details: outcome, settlement amount, timeline
- Links to case sites
- Suggested precedent pleadings/motions from similar cases

**Integration Points:**
- **SharePoint Search:** Query across all lists + documents
- **Graph API:** Search with KQL syntax
- **Runs in:** SharePoint page with search web part OR Azure Function with custom search UI

**MVP Implementation (low-friction):**
1. Create "Case Search" SharePoint page
2. Add search box web part configured to search CLARK list + all case site libraries
3. Apply managed property mappings:
   - InjuryType → RefinableString01
   - Venue → RefinableString02
   - OpposingCounsel → RefinableString03
4. Configure refiners (filters) on left: Injury Type, Venue, Outcome
5. Create custom display template to show:
   - Case number, client name, injury type, settlement amount, outcome
   - Link to case site
6. Add "Find Similar" button in case sites that pre-populates search with current case's injury type + venue

**Alternative (Azure Cognitive Search - more powerful):**
1. Index all case data + documents in Azure Cognitive Search
2. Use semantic ranking to find similar cases
3. Build custom UI with similarity score
4. More expensive but better results

**Key Risks + Mitigations:**
- **Risk:** Search results incomplete (new sites not indexed)
  - *Mitigation:* Continuous crawl enabled in SharePoint, force re-index weekly
- **Risk:** Inconsistent injury type tagging
  - *Mitigation:* Use choice field with controlled vocabulary, not free text
- **Risk:** Slow search (10+ second response)
  - *Mitigation:* Optimize query scope, index only required properties
- **Risk:** Sensitive data exposed in search previews
  - *Mitigation:* Configure security trimming, test with low-privilege user

**Operator UX:**
1. Open new case: slip-and-fall at grocery store in Cook County
2. Click "Find Similar Cases" button in case site
3. Search auto-populated: InjuryType = "Slip and Fall", Venue = "Cook County"
4. See 8 matching cases from last 3 years
5. Apply refiner: Outcome = "Settlement"
6. See 5 cases, settlement range $15k-$45k
7. Open highest settlement case, review pleadings for precedent
8. Download motion to dismiss from that case, adapt for current case

---

### **13. EXPENSE RECONCILER**

**Problem:** Tracking case expenses (court reporters, medical records, filing fees) against budgets requires manual spreadsheet updates. No alerts when over budget.

**Trigger:** 
- Invoice uploaded to case site
- Expense manually logged
- Monthly reconciliation

**Inputs:**
- Invoice PDF
- Expense type (reporter, medical records, filing, expert)
- Amount
- Case number

**Outputs:**
- Expense item in "Case Expenses" list (hub level)
- Case budget updated (total spent vs. budget)
- Alert if budget exceeded
- Monthly expense report per case
- Year-end tax export (CSV)

**Integration Points:**
- **Azure AI Document Intelligence:** Extract amount + vendor from invoice
- **SharePoint REST:** Update expenses list
- **Power BI:** Expense dashboard (optional)
- **Runs in:** Power Automate OR Azure Function

**MVP Implementation:**
1. Create "Case Expenses" list at hub:
   - CaseNumber (managed metadata)
   - ExpenseType (choice: Reporter, Medical, Filing, Expert, Other)
   - Vendor
   - Amount (currency)
   - InvoiceDate
   - InvoiceURL (link to PDF in case site)
   - BudgetRemaining (calculated per case)
2. Add "CaseBudget" column to CLARK list (currency)
3. Power Automate flow triggered when invoice uploaded to case site:
   - Extract amount from PDF (Azure AI or manual entry form)
   - Create item in Case Expenses list
   - Calculate total spent for case (sum all expenses)
   - Update BudgetRemaining in CLARK list
   - If BudgetRemaining < 0 → send alert
4. Monthly report flow:
   - Group expenses by case
   - Export to Excel, email to Mark
5. Year-end flow: export all expenses to CSV for accountant

**Key Risks + Mitigations:**
- **Risk:** Invoice amount extraction fails
  - *Mitigation:* Fallback to manual entry form, validate extracted amount
- **Risk:** Duplicate expense entries
  - *Mitigation:* Check invoice URL before creating, warn if duplicate
- **Risk:** Budget not set for case
  - *Mitigation:* Default budget = policy limits * 0.1, require manual review
- **Risk:** Vendor name inconsistent (typo)
  - *Mitigation:* Use choice field for common vendors, allow "Other" with manual entry

**Operator UX:**
1. Invoice PDF arrives via email
2. Save to case site "Invoices" library
3. Automated processing:
   - Amount extracted: $450
   - Vendor: "ABC Court Reporting"
   - Expense type: auto-detected as "Reporter"
4. Review entry in Case Expenses list, verify accuracy
5. If wrong, manually correct amount or type
6. Dashboard shows: Budget $5,000, Spent $2,100, Remaining $2,900
7. If over budget, receive alert: "BUDGET EXCEEDED: Smith v. Jones - Spent $5,200 / Budget $5,000"

---

### **14. EXHIBIT FINDER**

**Problem:** Locating specific exhibits (photos, medical records, expert reports) across 50+ cases is time-consuming. No centralized exhibit registry.

**Trigger:** Manual search

**Inputs:**
- Search keywords (exhibit description, type, case)
- Date range

**Outputs:**
- List of matching exhibits with:
  - Exhibit number
  - Description
  - Case number
  - Date
  - File location (link to document)
- Preview thumbnails for images
- Download all matching exhibits as ZIP

**Integration Points:**
- **SharePoint Search:** Full-text + metadata search
- **Graph API:** Get drive items across sites
- **Runs in:** SharePoint search page OR custom Azure Function UI

**MVP Implementation:**
1. Create "Exhibits" library in each case site (or centralized at hub)
2. Required metadata columns:
   - ExhibitNumber
   - ExhibitDescription
   - ExhibitType (Photo, Medical, Expert Report, Other)
   - RelatedTo (plaintiff, defendant, witness name)
3. Ensure all exhibits stamped with metadata when uploaded (use Doc Tagger skill)
4. Create search page:
   - Search box configured to search across all Exhibits libraries
   - Refiners: ExhibitType, CaseNumber, Date
   - Results display: thumbnail + metadata + download link
5. Add "Download Selected" button to get ZIP of checked items

**Alternative (more sophisticated):**
1. Maintain centralized "Exhibit Registry" list at hub
2. When exhibit uploaded to case site, create entry in registry with link
3. Search registry instead of searching across sites (faster)
4. Use Power Automate to sync exhibit uploads to registry

**Key Risks + Mitigations:**
- **Risk:** Exhibits not tagged with metadata
  - *Mitigation:* Enforce required fields on upload, integrate with Doc Tagger
- **Risk:** Duplicate exhibit numbers
  - *Mitigation:* Use format "CASEID-EXH-001", validate uniqueness
- **Risk:** Search slow across 50 sites
  - *Mitigation:* Use centralized registry approach OR scope search to specific cases
- **Risk:** Privileged exhibits exposed
  - *Mitigation:* Mark privileged in metadata, exclude from search unless user has permission

**Operator UX:**
1. Mark asks: "Find all photos of intersection from Smith case"
2. Go to Exhibit Search page
3. Enter keywords: "intersection photo Smith"
4. Apply filter: ExhibitType = Photo
5. See 12 matching results with thumbnails
6. Click exhibit to preview
7. Select 3 relevant photos, click "Download Selected"
8. Get ZIP file, share with Mark

---

### **15. CLIENT COMMUNICATION LOGGER**

**Problem:** Emails to/from clients aren't consistently saved to case files. Difficult to reconstruct conversation history.

**Trigger:** 
- Email sent to client (from Mark or you)
- Email received from client

**Inputs:**
- Email content
- Sender/recipient
- Timestamp
- Case number (extracted or manually tagged)

**Outputs:**
- Email saved to case site Correspondence library
- Metadata: CommunicationType (Client Email), Direction (Inbound, Outbound), Summary
- Timeline view of all client communications
- Searchable by keyword

**Integration Points:**
- **Graph API:** Monitor Mark's sent items + inbox
- **SharePoint REST:** Save email as MSG file
- **Runs in:** Azure Function OR Power Automate (flow on email send)

**MVP Implementation:**
1. Set up Graph webhook subscription for Mark's mailbox (sent items + inbox)
2. Azure Function processes each email:
   - Check if sender/recipient is client (match against Clients list)
   - Extract case number from subject OR prompt to tag if missing
   - Download email as MSG file
   - Upload to case site Correspondence library
   - Tag metadata: Direction, CommunicationType, DateSent
3. Handle attachments: save separately or embed in MSG
4. Create timeline view in case site showing all client communications

**Alternative (manual Outlook rule - lower tech):**
1. Outlook rule: emails from client addresses → BCC to case email address
2. Power Automate flow: when email arrives at case address, save to SharePoint
3. Less reliable but simpler

**Key Risks + Mitigations:**
- **Risk:** High volume triggers webhook throttling
  - *Mitigation:* Batch processing, delta query, cache client list
- **Risk:** Personal emails accidentally logged
  - *Mitigation:* Only process emails where client in To/From field
- **Risk:** Case number not in subject
  - *Mitigation:* Fuzzy matching on client name, manual review queue
- **Risk:** Sensitive content in emails
  - *Mitigation:* Apply same security as case site, never log to shared location

**Operator UX:**
1. Mark sends email to client: "Update on your case..."
2. Within 5 min, email auto-saved to case site (no action needed)
3. Check case site Correspondence library, see email listed
4. Timeline view shows chronological client communications
5. If email not auto-logged (case # missing), manually forward to case email address
6. System prompts to select case, saves to correct location

---

### **16. DISCOVERY DEADLINE MONITOR**

**Problem:** Discovery deadlines have complex 30/60/90 day cycles. Easy to miss interrogatory response deadlines, deposition notices, etc.

**Trigger:** 
- Complaint filing date entered
- Discovery request served (detected via Service Email Processor)
- Manual "Set Discovery Schedule" button

**Inputs:**
- Complaint filing date OR service date
- Discovery type (interrogatories, requests to produce, depositions)
- Jurisdiction (Illinois rules default)

**Outputs:**
- Discovery deadline schedule in Deadlines list:
  - Initial disclosures (30 days)
  - Interrogatory responses (28 days from service)
  - Deposition notice minimum (14 days)
  - Discovery cutoff (120 days before trial)
- Calendar events for each deadline
- Alerts 7 days before each deadline
- Status tracking (completed, pending, overdue)

**Integration Points:**
- **SharePoint REST:** Update Deadlines list
- **Graph API:** Create calendar events
- **Power Automate:** Alert emails
- **Runs in:** Power Automate flow

**MVP Implementation:**
1. Extend Deadlines list with "Discovery" category
2. Power Automate flow (manual trigger from CLARK list):
   - Get filing date
   - Calculate deadlines per Illinois rules:
     - Initial disclosures: filing date + 30 days
     - Discovery cutoff: trial date - 120 days (if trial set)
   - Create deadline items for each
   - Create calendar events
3. Second flow: when discovery request logged (via Service Email Processor)
   - Calculate response due: service date + 28 days
   - Create deadline
   - Create calendar reminder
4. Daily alert flow (same as Deadline Tracker Sync)

**Key Risks + Mitigations:**
- **Risk:** Trial date changes, discovery cutoff recalculates wrong
  - *Mitigation:* Detect trial date update, recalculate all dependent deadlines, notify
- **Risk:** Jurisdiction rules vary (federal vs. state)
  - *Mitigation:* Require jurisdiction field, validate against rule set
- **Risk:** Service date unknown
  - *Mitigation:* Use date logged as fallback, flag for review
- **Risk:** Complex calculations (holidays, extensions)
  - *Mitigation:* Build rule engine OR integrate with legal calendaring service

**Operator UX:**
1. New case filed, enter filing date in CLARK list: "2025-02-01"
2. Click "Set Discovery Schedule" button
3. See message: "Discovery schedule created. Initial disclosures due: March 3, 2025"
4. Review Deadlines list, see all discovery milestones
5. When discovery request received, Service Email Processor auto-adds response deadline
6. Receive alerts 7 days before each deadline
7. Mark as complete in Deadlines list when responses served

---

### **17. INVOICE GENERATOR (Enhanced)**

**Problem:** Creating invoices manually requires copying data from multiple sources, calculating totals, and formatting in Word/Excel.

**Trigger:** Manual "Generate Invoice" button in case site OR monthly billing schedule

**Inputs:**
- Case number
- Time entries (date, description, hours, rate)
- Expenses (from Case Expenses list)
- Billing period (start/end dates)
- Client billing address

**Outputs:**
- PDF invoice with:
  - Case details, billing period
  - Time entries table
  - Expenses table
  - Subtotals, tax, total due
  - Payment instructions
- Invoice saved to case site Billing library
- Invoice item created in "Invoices" list (hub level)
- Email sent to client with PDF attached
- Outstanding balance tracked in CLARK list

**Integration Points:**
- **SharePoint REST:** Get time entries, expenses
- **Word template OR PDF generation library:** Create invoice
- **Graph API:** Send email
- **Runs in:** Azure Function OR Power Automate with PDF generation

**MVP Implementation:**
1. Create "Time Entries" list at hub (or per case):
   - CaseNumber, Date, Description, Hours, Rate, Amount (calculated)
2. Build invoice template (Word with content controls OR HTML template for PDF conversion)
3. Power Automate flow (manual trigger):
   - Get case details from CLARK list
   - Get time entries for billing period (filter by date range)
   - Get expenses for period
   - Calculate totals
   - Populate Word template OR generate HTML → convert to PDF
   - Save PDF to Billing library
   - Create invoice item in Invoices list
   - Send email to client
4. Track payment status in Invoices list (Sent, Paid, Overdue)

**Key Risks + Mitigations:**
- **Risk:** Time entries missing or incorrect
  - *Mitigation:* Require time entry approval before invoicing, validate rates
- **Risk:** Duplicate invoicing
  - *Mitigation:* Mark time entries as "Billed" after inclusion, prevent re-billing
- **Risk:** Client email bounces
  - *Mitigation:* Validate email, CC yourself, log send status
- **Risk:** Tax calculation wrong
  - *Mitigation:* Use fixed tax rate OR integrate with accounting software

**Operator UX:**
1. End of month, go to case site
2. Click "Generate Invoice" button
3. Select billing period: "Feb 1-28, 2025"
4. Preview time entries (15 entries, 12.5 hours total)
5. Review expenses (3 items, $500 total)
6. Verify totals: $2,500 (time) + $500 (expenses) = $3,000
7. Click "Send to Client"
8. Invoice PDF saved to Billing library, email sent
9. Track payment status in Invoices list

---

### **18. MEDICAL RECORDS REQUEST AGENT**

**Problem:** Requesting medical records requires sending HIPAA-compliant letters to providers, tracking requests, following up.

**Trigger:** Manual "Request Medical Records" button

**Inputs:**
- Provider name + address
- Patient name + DOB
- Date range for records
- Authorization (signed by client)
- Case number

**Outputs:**
- HIPAA authorization letter generated
- Cover letter to provider
- Both saved to case site Medical library
- Tracking item in "Medical Requests" list
- Reminder set for follow-up (30 days)
- Status: Requested, Received, Overdue

**Integration Points:**
- **Word template:** HIPAA auth + cover letter
- **Graph API:** Send fax OR email (if provider accepts email)
- **SharePoint:** Medical Requests list
- **Runs in:** Power Automate flow

**MVP Implementation:**
1. Create "Medical Requests" list at hub:
   - CaseNumber, ProviderName, PatientName, RequestDate, ExpectedDate, Status, RecordsReceivedDate
2. Build templates:
   - HIPAA authorization form (patient signs)
   - Cover letter to provider
3. Power Automate flow (manual trigger from case site):
   - Get case + patient details
   - Populate templates
   - Save PDFs to Medical library
   - Create tracking item
   - Send via fax service (e-Fax API) OR email
   - Set reminder for 30 days out
4. Daily check flow:
   - Get requests where Status = Requested AND ExpectedDate < Today
   - Send follow-up fax/email
   - Update Status = Overdue
5. Manual update when records arrive

**Key Risks + Mitigations:**
- **Risk:** Provider doesn't respond
  - *Mitigation:* Automated follow-ups at 30, 45, 60 days
- **Risk:** Wrong provider address
  - *Mitigation:* Maintain provider contact list, validate before sending
- **Risk:** Authorization expired (1 year limit)
  - *Mitigation:* Date auth, alert if >11 months old
- **Risk:** Records arrive but not logged
  - *Mitigation:* Reminder to update status when received

**Operator UX:**
1. Click "Request Medical Records" in case site
2. Enter provider: "Memorial Hospital - 123 Main St"
3. Enter patient details: "John Smith, DOB 1/1/1980"
4. Select date range: "1/1/2020 - 12/31/2024"
5. Upload signed authorization PDF
6. Click "Send Request"
7. Documents generated + fax sent
8. Receive reminder in 30 days if not received
9. When records arrive, mark Status = Received, upload to Medical library

---

### **19. OPPOSING COUNSEL DATABASE**

**Problem:** No centralized repository of opposing counsel contact info, past interactions, settlement tendencies.

**Trigger:** 
- New case assigned with opposing counsel
- Manual entry after first contact

**Inputs:**
- Attorney name
- Firm name
- Contact info (email, phone, address)
- Case history (from CLARK list)
- Notes (settlement tendency, responsiveness, specialty)

**Outputs:**
- "Opposing Counsel" list at hub with all counsel info
- Auto-populate counsel info when creating new cases
- Dashboard showing:
  - Cases per counsel
  - Average settlement amount
  - Average case duration
- Alerts when counsel known to be difficult
- Search function to find counsel by name/firm

**Integration Points:**
- **SharePoint:** Opposing Counsel list
- **Power BI:** Dashboard (optional)
- **Runs in:** SharePoint list + calculated columns + flows

**MVP Implementation:**
1. Create "Opposing Counsel" list at hub:
   - AttorneyName, FirmName, Email, Phone, Address, Specialty, Notes, SettlementTendency (choice: Reasonable, Difficult, Very Difficult)
2. Add "OpposingCounselID" lookup column to CLARK list → links to Opposing Counsel list
3. Power Automate flow: when new case created
   - If opposing counsel already in list → auto-populate contact info
   - If new counsel → prompt to add to list
4. Create dashboard view:
   - Group cases by opposing counsel
   - Calculate metrics: avg settlement, avg duration, case count
5. Add notes section for free-text observations
6. Create alert flow: when difficult counsel assigned
   - Notify Mark: "Heads up: Jane Doe is representing plaintiff. Previous cases took avg 18 months."

**Key Risks + Mitigations:**
- **Risk:** Duplicate entries for same counsel
  - *Mitigation:* Search before adding new, merge duplicates quarterly
- **Risk:** Contact info outdated
  - *Mitigation:* Validate on use, update when emails bounce
- **Risk:** Subjective notes (bias)
  - *Mitigation:* Keep notes factual, avoid personal opinions
- **Risk:** Privacy concerns
  - *Mitigation:* Restrict access to internal team only

**Operator UX:**
1. New case: plaintiff represented by "Jane Doe, ABC Law"
2. Start typing "Jane Doe" in OpposingCounsel field
3. Auto-complete suggests existing entry
4. Select → contact info auto-populated
5. Review dashboard: Jane Doe has 5 prior cases, avg settlement $35k
6. Read notes: "Responsive to settlement discussions, prefers mediation"
7. Use this intel for case strategy
8. After case resolves, update notes with outcome

---

### **20. AUTOMATED STATUS REPORTS**

**Problem:** Mark needs weekly status updates on all active cases, but manually compiling takes 1+ hour.

**Trigger:** Scheduled weekly (Monday 8am)

**Inputs:**
- All active cases (from CLARK list)
- Recent activity: deadlines met, emails sent/received, documents filed, expenses

**Outputs:**
- HTML email report with:
  - Summary: # active cases, upcoming deadlines this week, overdue items
  - Section per case:
    - Case name, number
    - Status (discovery, mediation, trial prep)
    - Activity this week (auto-detected from lists)
    - Upcoming deadlines
    - Red flags (overdue, budget exceeded)
  - Action items highlighted
- PDF version saved to SharePoint
- Optionally: dashboard page with same info

**Integration Points:**
- **SharePoint REST:** Get all list data (cases, deadlines, expenses, time entries, correspondence)
- **Graph API:** Send email
- **Runs in:** Power Automate OR Azure Function

**MVP Implementation:**
1. Power Automate flow (scheduled, Mon 8am):
   - Get all CLARK list items where Status ≠ Closed
   - For each case:
     - Get deadlines (next 7 days + overdue)
     - Get recent correspondence (last 7 days)
     - Get expenses (total spent vs budget)
     - Get recent time entries
     - Flag red flags: overdue deadlines, over budget, no activity >30 days
   - Build HTML email:
     - Summary section at top
     - Case sections below
     - Red flags highlighted in red
   - Send to Mark (+ you if desired)
   - Save as PDF to Reports library
2. Create SharePoint dashboard page with same data (refresh on demand)

**Key Risks + Mitigations:**
- **Risk:** Email too long (50+ cases)
  - *Mitigation:* Paginate OR filter to cases with activity only
- **Risk:** Inaccurate data (stale deadlines)
  - *Mitigation:* Data hygiene checks before generating report
- **Risk:** Missing critical info
  - *Mitigation:* Template includes all key fields, review quarterly
- **Risk:** Mark doesn't read (email overload)
  - *Mitigation:* Make it scannable, use color coding, keep <2 pages

**Operator UX:**
1. Monday 8am, automated report sent (no action needed)
2. Mark opens email, sees summary:
   - 23 active cases
   - 5 deadlines this week
   - 2 overdue items (red flag)
3. Scroll to case sections, review activity
4. Click links to case sites for details
5. Follow up on red flags
6. If questions, reply to email OR check dashboard page
7. Report archived to SharePoint for reference

---

## TOP 5 PRIORITY IMPLEMENTATIONS

Based on impact, feasibility, and your constraints:

### **1. NEW MATTER SITE GENERATOR**
**Why first:** Currently blocks operations. High ROI (saves 20+ min per case). Foundation for other automations. You have Azure Function infrastructure ready.

**Effort:** Medium (1-2 weeks)  
**Impact:** Critical  
**Dependencies:** None  
**Quick win:** Get this working with basic template, iterate on libraries/permissions later.

---

### **2. CASE METADATA EXTRACTOR (Email Parser v2)**
**Why second:** Already partially built. Eliminates manual data entry errors. Direct integration with #1 (auto-provisions site after extraction).

**Effort:** Medium (1-2 weeks)  
**Impact:** High  
**Dependencies:** #1 (optional but synergistic)  
**Quick win:** Start with First Chicago emails only, expand to other carriers later.

---

### **3. CALENDAR-TO-CASE DASHBOARD**
**Why third:** You specifically want this. Low-friction Power Automate implementation. Immediate visibility for Mark. Reduces "where's the case file" questions.

**Effort:** Low (3-5 days)  
**Impact:** Medium-High  
**Dependencies:** CLARK list structure stable  
**Quick win:** Build MVP with simple matching (court # in subject), refine fuzzy logic later.

---

### **4. DEADLINE TRACKER SYNC**
**Why fourth:** Prevents malpractice. Complements #3 (calendar integration). Foundational for other deadline-related skills (#6, #16).

**Effort:** Medium (1 week)  
**Impact:** High (risk mitigation)  
**Dependencies:** #3 (calendar access)  
**Quick win:** Start with manual deadline entry, add auto-extraction from emails later.

---

### **5. INCOMING DOC TAGGER**
**Why fifth:** Reduces future search friction. Makes other skills work better (Exhibit Finder, Cross-Case Search). Can start simple (keywords) and add AI later.

**Effort:** Medium-High (2 weeks)  
**Impact:** Medium (compounds over time)  
**Dependencies:** Doc libraries structured consistently  
**Quick win:** Tag DocumentType only, add more metadata fields iteratively.

**NOT in top 5 (but valuable):**
- **Pleading Template Filler** (#7) - High value but Mark may prefer manual review for now
- **Settlement Authority Tracker** (#10) - Important but lower urgency than deadline tracking
- **Expense Reconciler** (#13) - Useful but not critical path

---

## DATA HYGIENE CONVENTIONS

For automations to work reliably, enforce these conventions:

### **Naming Standards**

**Case Numbers (CourtNumber managed metadata):**
- Format: `YYYY-L-#####` (e.g., `2024-L-01234`)
- Always include in email subjects, calendar events, document filenames
- Use as primary key for linking data

**Claim Numbers:**
- Format: `CARRIER-######` (e.g., `FCI-123456`)
- Stored in CLARK list, used for insurance lookup

**Site URLs:**
- Format: `/sites/Case-YYYY-L-#####` (e.g., `/sites/Case-2024-L-01234`)
- Consistent structure enables URL generation from case number

**Calendar Event Subjects:**
- Format: `[Event Type] - [Case Name] - [Court #] - [Date/Time]`
- Example: `Hearing - Smith v. Jones - 2024-L-01234 - 3/15/25 9am`
- Enables automated matching

### **Required Metadata Fields (CLARK List)**

**Tier 1 (mandatory for site provisioning):**
- CaseNumber (managed metadata, unique)
- ClientName (text)
- CaseType (choice: MVA, Premises, General PI)
- DateFiled (date)
- Status (choice: Intake, Discovery, Mediation, Trial Prep, Closed)

**Tier 2 (required for full automation):**
- ClaimNumber (text)
- PolicyLimits (currency)
- OpposingCounselID (lookup to Opposing Counsel list)
- AdjusterID (lookup to Adjusters list)
- CourtVenue (choice: Cook County, DuPage, etc.)

**Tier 3 (optional but recommended):**
- DateOfLoss (date)
- IncidentType (choice: Auto, Slip-Fall, Dog Bite, etc.)
- AssignedAttorney (person field, default: Mark)
- CaseBudget (currency)
- CaseSiteURL (hyperlink, auto-populated by Site Generator)

### **Document Library Standards**

**Standard Libraries per Case Site:**
1. **Pleadings** - Complaints, answers, motions
2. **Discovery** - Requests, responses, depositions
3. **Correspondence** - Emails, letters
4. **Medical** - Records, bills, expert reports
5. **Expert** - Reports, disclosures
6. **Billing** - Invoices, time sheets

**Required Columns (all libraries):**
- DocumentType (choice field, specific to library)
- DateReceived (date)
- CourtNumber (managed metadata, inherited from site)
- Tags (keywords for search)

### **Workflow Conventions**

**Email Subject Lines:**
- Always include court number: `[2024-L-01234] - Subject`
- For service: "SERVED - [Document Type] - [Court #]"
- For filing: "FILED - [Document Type] - [Court #]"

**Status Transitions (CLARK list):**
- Intake → Discovery (when answer filed)
- Discovery → Mediation (when mediation scheduled)
- Mediation → Trial Prep (if no settlement)
- Trial Prep → Closed (after resolution)
- Trigger automation on status changes (e.g., close deadlines when case closed)

**Case Closure Checklist:**
1. Mark all deadlines complete
2. Move to "Closed Cases" view
3. Archive correspondence
4. Export financial records
5. Set site to read-only (preserve for 7 years per IL rules)

### **Validation Rules**

**Prevent common errors:**
- Case number must be unique (enforce in list)
- Policy limits cannot be $0 (validation)
- Service date cannot be future (validation)
- Date of loss must be before file date (validation)
- Court number format validated with regex

**Data Quality Checks (monthly):**
- Cases missing site URL → run Site Generator
- Cases with Status = Intake >60 days → flag for review
- Deadlines >180 days old still "Pending" → archive or complete
- Opposing counsel duplicates → merge
- Orphaned documents (no case link) → review and assign

---

## REUSABLE SKILL DEFINITION TEMPLATE

Use this template for each new skill you implement:

```markdown
# SKILL: [Skill Name]

## Overview
**Purpose:** [1-2 sentence problem statement]
**Value:** [Quantified benefit - time saved, errors prevented, etc.]
**Owner:** [Who maintains this - you, Mark, external vendor]

## Technical Specification

### Trigger(s)
- [ ] Manual button/command
- [ ] Scheduled (frequency: ____)
- [ ] Webhook (event: ____)
- [ ] List item created/modified
- [ ] Email received
- [ ] File uploaded

### Inputs
| Input | Type | Source | Required | Validation |
|-------|------|--------|----------|------------|
| [Field name] | [Text/Date/Number] | [List/Email/User] | Yes/No | [Rule] |

### Outputs
| Output | Type | Destination | Format |
|--------|------|-------------|--------|
| [What created] | [File/List item/Email] | [Where saved] | [Structure] |

### Integration Points
**Microsoft Graph API:**
- Endpoint: `[METHOD] /[resource]`
- Permission: `[Scope]`
- Throttling: [Limit]

**SharePoint REST/PnP:**
- List: `[List name]`
- Operation: `[CRUD action]`

**Third-Party:**
- Service: `[Name]`
- API: `[Endpoint]`

### Runtime Environment
- [ ] Azure Function (HTTP/Timer/Queue triggered)
- [ ] Power Automate (Cloud flow)
- [ ] Logic App
- [ ] SPFx Web Part
- [ ] Local script (PowerShell/Python/Node)

**Authentication:**
- Method: [Cert-based / OAuth / Managed Identity]
- App Registration ID: `[GUID]`
- Certificate: `[Thumbprint or Key Vault reference]`

## Implementation

### Dependencies
**Required:**
- Microsoft 365 licenses: [E3/E5/Business Premium]
- Azure resources: [Function App, Storage Account, etc.]
- App permissions: [List required Graph/SharePoint scopes]
- Lists/libraries: [CLARK, Deadlines, etc.]

**Optional:**
- Power BI license (for dashboards)
- Azure AI services (for ML/OCR)

### MVP Steps
1. [Step 1 - setup]
2. [Step 2 - core logic]
3. [Step 3 - integration]
4. [Step 4 - testing]
5. [Step 5 - deployment]

**Estimated Effort:** [Hours/Days]

### Testing Plan
**Test Cases:**
1. Happy path: [Expected input → output]
2. Edge case: [Unusual input → expected behavior]
3. Error case: [Invalid input → error handling]

**Test Data:**
- Use case: `[Case #]`
- Test inputs: `[Sample data]`

### Deployment Checklist
- [ ] Code committed to repo
- [ ] App registration configured
- [ ] Permissions granted
- [ ] Secrets stored in Key Vault
- [ ] Monitoring/logging enabled
- [ ] Documentation updated
- [ ] User training completed
- [ ] Rollback plan documented

## Operations

### Risk Register
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| [What could go wrong] | L/M/H | L/M/H | [How to prevent/handle] |

### Monitoring
**Success Metrics:**
- [Metric 1]: [Target value]
- [Metric 2]: [Target value]

**Alerts:**
- Failure rate >5% → email
- Execution time >30s → log warning
- [Custom alert]

**Logs:**
- Location: [App Insights / SharePoint list]
- Retention: [Days]

### Maintenance
**Scheduled Tasks:**
- Daily: [Check health]
- Weekly: [Review logs]
- Monthly: [Data cleanup]
- Quarterly: [Template updates]

**Known Issues:**
- [Issue 1]: [Workaround]

## User Guide

### Operator Workflow
**Step-by-step:**
1. [Action] → [Result]
2. [Action] → [Result]
3. Verify: [Check this]

**Troubleshooting:**
- **Problem:** [Symptom]
  - **Solution:** [Fix]

### Support
**First contact:** [You / IT / Vendor]
**Escalation:** [Mark / Azure support]
**Documentation:** [Link to detailed guide]

---

**Version:** 1.0  
**Last Updated:** [Date]  
**Change Log:**
- [Date]: [Change description]
```

---

## IMPLEMENTATION ROADMAP

### Phase 1: Foundation (Weeks 1-4)
**Focus:** Core infrastructure + highest-value automations

**Week 1-2:**
- Implement #1: New Matter Site Generator
- Set up Azure Function App monitoring
- Document site template structure

**Week 3-4:**
- Implement #2: Case Metadata Extractor
- Connect to Site Generator (end-to-end case intake)
- Test with 5 real assignment emails

**Deliverable:** Automated case intake from email → provisioned site in <2 min

---

### Phase 2: Visibility (Weeks 5-7)
**Focus:** Connect calendar, track deadlines

**Week 5:**
- Implement #3: Calendar-to-Case Dashboard
- Create matching algorithm
- Build dashboard page

**Week 6:**
- Implement #4: Deadline Tracker Sync
- Set up Graph webhook for calendar
- Build alert flows

**Week 7:**
- Integrate #3 + #4 (calendar events → deadlines)
- Add "Daily Docket" view for Mark
- Test alert delivery

**Deliverable:** Mark has daily visibility into hearings + deadlines

---

### Phase 3: Intelligence (Weeks 8-11)
**Focus:** Document automation + search

**Week 8-9:**
- Implement #5: Incoming Doc Tagger
- Start with keyword-based classification
- Add OCR for scanned PDFs

**Week 10:**
- Implement #12: Cross-Case Search Agent
- Configure search page
- Test similarity matching

**Week 11:**
- Implement #19: Opposing Counsel Database
- Migrate existing counsel data
- Build dashboard

**Deliverable:** Searchable document repository + case intelligence

---

### Phase 4: Expansion (Weeks 12+)
**Focus:** Add remaining skills based on feedback

**Priority:**
- #7: Pleading Template Filler (if Mark wants automation)
- #10: Settlement Authority Tracker (financial oversight)
- #20: Automated Status Reports (weekly visibility)

**Iterate:**
- Refine existing skills based on usage
- Add new skills as needs emerge
- Optimize performance (speed, accuracy)

---

## NEXT STEPS

**Immediate actions:**

1. **Review this proposal** - Flag any skills that don't fit or need modification

2. **Prioritize** - Confirm top 5 or adjust based on current pain points

3. **Prepare environment:**
   - Verify Azure Function App working
   - Audit CLARK list structure (add missing columns)
   - Document current site template (libraries, permissions)
   - Test Graph API access with cert auth

4. **Pick one skill to prototype** - I recommend #1 (Site Generator) for quick win

5. **Set success criteria** - Define "done" for MVP (e.g., "creates site with 5 libraries + correct permissions in <60 seconds")

**Questions for you:**

- Do any of these 20 skills overlap with existing (broken) automation you want to replace?
- Are there workflows I missed that cause daily friction?
- What's your appetite for SPFx vs. sticking with Power Automate + Azure Functions?
- Should some skills share infrastructure (e.g., one webhook handler for multiple events)?

---

I've provided concrete, implementation-ready specifications. Each skill has clear inputs/outputs, Graph API endpoints, risk mitigations, and operator UX. The top 5 prioritization focuses on unblocking you first, then adding intelligence.

Ready to dive into implementing #1 (Site Generator) or want to discuss trade-offs?
