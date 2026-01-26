# Browser Security Operator Playbook

**Version:** 1.0  
**Last Updated:** 2026-01-26  
**Scope:** Cross-Platform Browser Security Operations (macOS, Windows, Linux)

---

## FIRST QUESTIONS (ANSWER ALL BEFORE PROCEEDING)

**1. Operating System & Browser**
- macOS / Windows / Linux (specify distro)
- Chrome / Brave / Edge / Vivaldi / Chromium / Other

**2. Incognito Behavior**
- Does the unwanted behavior **completely disappear** in Incognito mode? (YES/NO)

**3. Policy Status**
- Navigate to `chrome://policy`
- Is the page **empty** or does it list policies? (EMPTY / HAS POLICIES)
- If HAS POLICIES: Export JSON and attach

**4. Management Status**
- Navigate to `chrome://management`
- Does it say "Your browser is managed" or "not managed"? (MANAGED / NOT MANAGED)

**5. Symptom Description**
- DOM injection (describe elements/screenshot)
- Redirect (from → to)
- Locked settings (which settings)
- Unwanted extension (name/ID)
- Other (specify)

---

## ROLE & SCOPE

### Mission Statement
Aggressively detect, isolate, and remove browser-based threats through systematic investigation and proof-based elimination.

### Boundaries
- **IN SCOPE:** Browser profiles, extensions, policies, storage, service workers, certificate stores
- **OUT OF SCOPE:** OS kernel modifications, network stack manipulation, system-level rootkits

### Operating Rules
1. **Blunt execution** — No speculation, only verifiable facts
2. **Technical precision** — Command-first approach with exact syntax
3. **Proof-oriented** — Every action requires verification step
4. **Minimal assumptions** — User provides evidence, not descriptions

---

## TRIAGE PROTOCOL

### Decision Tree

```
┌─ Symptom Present in NORMAL window?
│
├─ NO → False alarm / user error
│
├─ YES → Symptom Present in INCOGNITO?
   │
   ├─ NO → EXTENSION-BASED THREAT
   │        → Go to PRIMARY PLAYBOOK
   │
   ├─ YES → Check chrome://policy
      │
      ├─ HAS POLICIES → POLICY/PROFILE HIJACK
      │                  → Go to SECONDARY PLAYBOOK
      │
      └─ EMPTY → Check chrome://management
         │
         ├─ MANAGED → ENTERPRISE/MDM CONTROL
         │            → Go to SECONDARY PLAYBOOK (Step 3)
         │
         └─ NOT MANAGED → SITE-LEVEL PERSISTENCE
                          → Go to TERTIARY PLAYBOOK
```

### Reproduction Methodology

**Standard Test Sequence:**
1. **Normal window** — Document exact behavior (screenshot/video)
2. **Incognito mode** — Same actions, observe differences
3. **Fresh profile** — chrome://settings/people → "Add person" → test again
4. **Post-restart** — Reboot browser completely, verify persistence

### Classification Framework

| Symptom Type | Incognito Clean? | Policy Page | Likely Cause |
|--------------|------------------|-------------|--------------|
| Unwanted extension visible | YES | Empty | Extension installed via sync/policy |
| DOM injection on websites | NO | Empty | Malicious extension |
| Settings locked/greyed out | Either | Has entries | Enterprise policy or malware policy |
| Redirect on specific sites | YES | Empty | Service worker hijack |
| Certificate warnings | Either | Either | Certificate store manipulation |
| Persistent popups | NO | Empty | Extension-based adware |

---

## PRIMARY PLAYBOOK — Extension & DOM Injection

**Verdict:** Extension is modifying page content or injecting unwanted behavior

**Use When:**
- Symptom disappears completely in Incognito mode
- chrome://policy is empty
- chrome://management shows "not managed"

---

### Step 1 — Prove Extension Involvement

**WHY:** Establish baseline that extensions are the attack vector

**EXACT ACTION:**
```
1. Open affected site in normal window → observe symptom
2. Open same site in Incognito window (Ctrl+Shift+N / Cmd+Shift+N)
3. If symptom gone in Incognito → extension confirmed
```

**EXPECTED RESULT:** Symptom completely absent in Incognito

**HOW TO VERIFY:**
- [ ] Side-by-side comparison (normal vs Incognito)
- [ ] Screenshot both windows showing URL bar (proves mode)

**Escalation Trigger:** If symptom persists in Incognito → go to SECONDARY PLAYBOOK

---

### Step 2 — Identify Offending Extension (Binary Search)

**WHY:** Pinpoint exact extension causing behavior

**EXACT ACTION:**
```
1. Navigate to chrome://extensions
2. Enable "Developer mode" (top-right toggle)
3. Screenshot current state (shows all IDs + sources)
4. Count total extensions → N
5. Disable ALL extensions (toggle each one OFF)
6. Verify symptom gone
7. Enable first N/2 extensions
8. Test symptom:
   - If present → offender in first half, disable all, enable first N/4
   - If absent → offender in second half, enable remaining
9. Repeat binary search until single extension identified
```

**EXPECTED RESULT:** Symptom appears/disappears based on specific extension state

**HOW TO VERIFY:**
- [ ] Reproduce symptom with ONLY suspect extension enabled
- [ ] Symptom absent with suspect extension disabled
- [ ] Re-enable suspect extension → symptom returns

---

### Step 2b — Validate Threat Status (Threat Intelligence Integration)

**WHY:** Confirm malicious nature before removal

**EXACT ACTION:**
```
1. Copy extension ID from chrome://extensions (28-32 character alphanumeric)
   Example: "abcdefghijklmnopqrstuvwxyz123456"

2. Web search: "chrome extension [ID] malware"

3. Automated analysis:
   - Visit: https://crxcavator.io/report/[EXTENSION_ID]
   - Check risk score and flagged permissions

4. Chrome Web Store check (if listed):
   - Search Web Store by ID
   - Read recent reviews (sort by "Most recent")
   - Check for malware/adware complaints

5. Document findings for post-cleanup report
```

**EXPECTED RESULT:** Risk assessment from multiple sources

**Evidence Required:**
- [ ] Screenshot of crxcavator.io report
- [ ] Screenshot of Web Store reviews (if available)
- [ ] Search results showing malware mentions

---

### Step 3 — Remove and Verify

**WHY:** Eliminate threat and confirm complete removal

**EXACT ACTION:**
```
1. Navigate to chrome://extensions
2. Locate malicious extension by ID
3. Click "Remove" button
4. Confirm removal in dialog
5. Restart browser completely (quit and reopen)
6. Verify extension not present in chrome://extensions
7. Test original symptom on affected site
```

**EXPECTED RESULT:** Extension absent, symptom eliminated

**Verification Checklist:**
- [ ] Normal window test — symptom absent
- [ ] Incognito test — symptom still absent
- [ ] Post-restart confirmation — extension not reinstalled
- [ ] chrome://extensions — confirm ID not present

**Escalation Trigger:** If extension reappears after restart → go to SECONDARY PLAYBOOK (Step 4)

---

## SECONDARY PLAYBOOK — Policy/Profile Hijack

**Verdict:** Browser controlled by enterprise policy, MDM, or account sync hijacking

**Use When:**
- chrome://policy shows entries
- chrome://management shows "managed"
- Extensions reinstall after removal

---

### Step 1 — Policy Inspection

**WHY:** Identify policy source and enforced settings

**EXACT ACTION:**
```
1. Navigate to chrome://policy
2. Click "Export policies as JSON" button (save file)
3. Click "Reload policies" button
4. Identify enforced policies:
   - ExtensionInstallForcelist (forced extension installs)
   - ExtensionInstallBlacklist (extension blocks)
   - HomepageLocation (locked homepage)
   - URLBlocklist/URLAllowlist (site restrictions)
   - ProxySettings (proxy hijacking)

5. Note "Source" column for each policy:
   - "Platform" = system-level (registry/plist/config file)
   - "Cloud" = Google account policy
   - "CBCM" = Chrome Browser Cloud Management
```

**Evidence Required:**
- [ ] Full-page screenshot of chrome://policy
- [ ] Exported JSON file

**Expected Result:** List of enforced policies with sources

---

### Step 2 — Management Status Check

**WHY:** Determine if browser is under enterprise control

**EXACT ACTION:**
```
1. Navigate to chrome://management
2. Read full message:
   - "Your browser is managed by [organization]" → enterprise/MDM control
   - "Your browser isn't managed" → false alarm (go to Step 4)

3. If managed, check chrome://policy → "Precedence" column:
   - Highest precedence = actual enforcer
   - Lower precedence policies may be overridden

4. For managed browsers with unwanted extensions:
   - Document managing organization name
   - Prepare for professional IT intervention
```

**Evidence Required:**
- [ ] Screenshot of chrome://management page
- [ ] Note organization name if shown

---

### Step 3 — Extension Install Source Analysis

**WHY:** Identify how extension persists (policy vs sync vs local)

**EXACT ACTION:**
```
1. Navigate to chrome://extensions
2. Enable "Developer mode"
3. For each suspicious extension, check "Installed by" field:
   - "Installed by enterprise policy" → policy-enforced
   - "Installed by [domain]" → sync or admin
   - No message → user-installed

4. If policy-enforced, cross-reference with chrome://policy:
   - Find ExtensionInstallForcelist policy
   - Match extension ID in policy JSON
   - Note policy source (Platform/Cloud/CBCM)
```

**Expected Result:** Clear attribution of installation method

**Platform-Specific Policy Removal:**

See **Windows Group Policy Check** and **Linux System Policy Check** sections below.

---

### Step 4 — Check Sync Persistence (Account Hijacking Detection)

**WHY:** Extensions may persist through compromised Google account sync

**EXACT ACTION:**
```
1. Navigate to chrome://settings/syncSetup/advanced
2. Check if "Extensions" toggle is ENABLED
3. If enabled and unwanted extension keeps reappearing:

   a. Sign out of Chrome account:
      - chrome://settings/people
      - Click "Turn off" under [your email]
      - Confirm sign-out

   b. Remove malicious extension:
      - chrome://extensions → Remove
      - Restart browser

   c. Verify extension stays removed:
      - Wait 5 minutes
      - Check chrome://extensions
      - If still absent → sync was the vector

   d. Before re-enabling sync, audit synced data:
      - Visit: https://myaccount.google.com
      - Navigate to "Security" → "Third-party apps with account access"
      - Revoke any suspicious apps
      - Navigate to "Data & privacy" → "Download your data"
      - Check synced Chrome extensions list

4. If extension reappears even when signed out → policy-based (return to Step 1)
```

**Evidence Required:**
- [ ] Screenshot of sync settings before sign-out
- [ ] Screenshot of myaccount.google.com third-party apps
- [ ] Extension list before/after sign-out

**Verification Checklist:**
- [ ] Extension removed while signed out
- [ ] Browser restarted
- [ ] Extension does not reinstall
- [ ] Account audit completed

**Escalation Trigger:** If extension reinstalls while signed out → system-level policy (see Platform-Specific sections)

---

## TERTIARY PLAYBOOK — Site-Level Persistence

**Verdict:** Website-specific storage or service worker hijacking

**Use When:**
- Symptom persists in Incognito mode
- chrome://policy is empty
- chrome://management shows "not managed"
- Symptom is site-specific, not browser-wide

---

### Step 1 — Service Worker Inspection

**WHY:** Rogue service workers can intercept requests and inject content

**EXACT ACTION:**
```
1. Navigate to chrome://serviceworker-internals
2. Locate affected domain in list
3. For each service worker from suspicious domain:
   - Note "Script URL"
   - Note "Status" (running/stopped)
   - Click "Unregister" button
   - Confirm in dialog

4. Alternative: Unregister via DevTools:
   - F12 → "Application" tab → "Service Workers"
   - Click "Unregister" for each worker
```

**Evidence Required:**
- [ ] Screenshot of chrome://serviceworker-internals showing domain
- [ ] Copy service worker script URL

**Expected Result:** Service workers removed for affected origin

---

### Step 2 — Site Storage Clearing

**WHY:** Malicious data in LocalStorage/IndexedDB may persist behavior

**EXACT ACTION:**
```
1. Navigate to affected site
2. Open DevTools (F12)
3. Go to "Application" tab
4. Expand "Storage" section on left
5. Clear all storage types for domain:

   - Local Storage → right-click domain → Clear
   - Session Storage → right-click domain → Clear
   - IndexedDB → right-click database → Delete database
   - Cookies → right-click domain → Clear all
   - Cache Storage → right-click cache → Delete

6. Alternative bulk method:
   - Application tab → "Clear storage" section
   - Check all boxes under "Storage"
   - Click "Clear site data" button
```

**Expected Result:** All origin-specific storage cleared

**Verification:**
- [ ] Refresh page (Ctrl+Shift+R / Cmd+Shift+R for hard reload)
- [ ] Check symptom disappeared
- [ ] Check Application tab → all storage types empty

---

### Step 3 — Origin-Specific Cleanup

**WHY:** Complete removal of site permissions and data

**EXACT ACTION:**
```
1. Navigate to chrome://settings/content/all
2. Search for affected domain
3. Click domain → "Clear data" button
4. Confirm removal

5. Check permissions reset:
   - chrome://settings/content/notifications → remove domain
   - chrome://settings/content/popups → remove domain
   - chrome://settings/content/javascript → remove domain blocks

6. Clear browsing data for specific timeframe:
   - chrome://settings/clearBrowserData
   - "Advanced" tab
   - Select time range when issue started
   - Check: Cookies, Cached images, Hosted app data
   - Click "Clear data"
```

**Verification Checklist:**
- [ ] Domain not listed in chrome://settings/content/all
- [ ] No stored permissions for domain
- [ ] Symptom absent after revisiting site
- [ ] Site appears in "fresh state" (asks for permissions again)

**Escalation Trigger:** If symptom persists after complete storage wipe → go to ESCALATION Level 2

---

## EVIDENCE COLLECTION STANDARDIZATION

### Protocol Overview
Collect proof for every diagnosis step to enable reproducible verification.

---

### chrome://policy Evidence

**Method 1 — JSON Export (Preferred):**
```
1. Navigate to chrome://policy
2. Click "Export policies as JSON" button
3. Save file as: chrome-policy-export-[YYYY-MM-DD].json
4. Share file for analysis
```

**Method 2 — Screenshot:**
```
1. Navigate to chrome://policy
2. Ensure full page visible (scroll to show all entries)
3. Take full-page screenshot (F12 → Ctrl+Shift+P → "Capture full size screenshot")
4. Annotate with date/time
```

**What to Capture:**
- Policy name column
- Value column
- Source column
- Precedence column

---

### chrome://extensions Evidence

**Required State:**
```
1. Navigate to chrome://extensions
2. Enable "Developer mode" toggle (top-right)
3. Expand all extension cards (click each one)
```

**What to Capture:**
- Extension name
- Extension ID (alphanumeric string)
- Version number
- "Inspect views" links (indicates active content scripts)
- "Permissions" list (expanded)
- "Installed by" field (if present)
- "Allow in Incognito" toggle state

**Screenshot Format:**
- One screenshot per extension (full card visible)
- Ensure ID is readable
- Show "Details" page if permissions are suspicious

---

### DOM Injection Evidence

**DevTools Console Method:**
```javascript
// Copy entire HTML (truncate if too long)
console.log(document.documentElement.outerHTML.substring(0, 5000));

// Find injected elements (common patterns)
console.log('Injected scripts:', document.querySelectorAll('script:not([src])'));
console.log('Injected iframes:', document.querySelectorAll('iframe'));
console.log('Suspicious divs:', document.querySelectorAll('[id*="ad"], [class*="popup"]'));

// Check for content script injection markers
console.log('Chrome extension contexts:', window.chrome?.runtime);
```

**Screenshot Method:**
```
1. F12 → "Elements" tab
2. Locate injected elements (unusual <div>, <script>, <iframe>)
3. Right-click element → "Capture node screenshot"
4. Also screenshot element's attributes (show data-* and id values)
```

**What to Capture:**
- Element type and ID/class names
- Parent element hierarchy
- Any data attributes containing URLs or IDs
- Network requests triggered by element (F12 → Network tab)

---

### Service Worker Evidence

**Inspection Method:**
```
1. Navigate to chrome://serviceworker-internals
2. Find affected origin in list
3. Copy table data for origin:
   - Registration ID
   - Scope URL
   - Script URL
   - Status (running/stopped/starting)
   - Running worker count
```

**Alternative - DevTools Method:**
```
1. On affected site, open F12
2. Application tab → Service Workers
3. Screenshot showing:
   - Source (script file)
   - Status
   - Update on reload checkbox
   - "Push" and "Sync" event options
```

**What to Capture:**
- Service worker script URL (may reveal extension ID)
- Scope (which URLs it intercepts)
- Status and lifecycle state
- Update timestamp

---

### Verification Screenshots (Before/After)

**Standard Format:**
```
Before:
- Screenshot of symptom (with URL bar visible)
- Screenshot of chrome://extensions (with malicious extension present)
- Screenshot of chrome://policy (if applicable)

After:
- Screenshot of same page (symptom absent)
- Screenshot of chrome://extensions (extension removed)
- Screenshot of chrome://policy (empty or policy removed)
- Screenshot of browser restart verification
```

**Timestamp Requirement:**
- Include system clock in screenshot (Windows: notification area, macOS: menu bar)
- Or screenshot chrome://about page showing version + build date

---

## VALIDATION CHECKLIST

Use this checklist after ANY remediation action:

### Post-Removal Verification

- [ ] **Normal window test**
  - Navigate to affected site/trigger symptom scenario
  - Symptom completely absent
  - No unexpected popups, redirects, or DOM modifications

- [ ] **Incognito test**
  - Open Incognito window (Ctrl+Shift+N / Cmd+Shift+N)
  - Navigate to affected site
  - Behavior same as normal window (no symptom)

- [ ] **Post-restart confirmation**
  - Completely quit browser (not just close window)
  - Reopen browser
  - Check chrome://extensions → malicious extension not reinstalled
  - Revisit affected site → symptom still absent

- [ ] **Policy confirmation**
  - Navigate to chrome://policy
  - Click "Reload policies" button
  - Confirm policy page empty OR no malicious policies listed

- [ ] **Extension list audit**
  - chrome://extensions → Developer mode ON
  - Review each extension:
    - Recognize the extension name/purpose
    - Check "Installed by" field (user vs policy)
    - Verify "Allow in Incognito" is OFF for untrusted extensions

- [ ] **Permissions audit**
  - chrome://settings/content/all
  - Check for unknown domains with stored permissions
  - Remove any suspicious entries

- [ ] **Management status**
  - chrome://management
  - Confirm "Your browser isn't managed" (unless legitimately managed)

### Continuous Monitoring (24-48 hours)

- [ ] Browser startup behavior (no unexpected tabs)
- [ ] New tab page unchanged
- [ ] Search engine not hijacked (chrome://settings/searchEngines)
- [ ] Homepage not locked (chrome://settings)
- [ ] Extensions list unchanged (no new unwanted extensions)

---

## ESCALATION CONDITIONS & RECOVERY

### Escalation Decision Matrix

| Condition | Level |
|-----------|-------|
| Symptom persists in fresh profile | Level 1 → Level 2 |
| Extension reinstalls within minutes of removal | Level 2 → Level 3 |
| Settings revert immediately after changes | Level 3 → Level 4 |
| Browser won't stay uninstalled | Level 4 |
| System certificates re-inject automatically | Level 4 |

---

## ESCALATION ACTIONS (in order of severity)

### Level 1: Isolated Profile Test

**Purpose:** Determine if issue is profile-specific or browser-wide

**Verdict Confidence:** 85% — Isolates profile corruption vs system-level issues

**Immediate Actions:**

1. **Create test profile** — WHY → Isolate corrupted profile → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Navigate to chrome://settings/people
   - Click "Add person" button
   - Create profile named "TEST-CLEAN-[DATE]"
   - Do NOT sign in to Google account
   - Expected: Fresh profile with zero extensions
   - Verify: chrome://extensions shows empty list
   ```

2. **Reproduce symptom in clean profile** — WHY → Determine if browser-wide → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Navigate to affected site in TEST-CLEAN profile
   - Perform same actions that triggered symptom
   - Expected: Symptom absent (confirms profile-specific)
   - Verify: Side-by-side comparison (old profile vs TEST-CLEAN)
   ```

3. **Migrate critical data** — WHY → Preserve user data → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Export bookmarks from old profile:
     chrome://bookmarks → ⋮ (menu) → "Export bookmarks" → Save HTML file
   - Import to TEST-CLEAN profile:
     chrome://bookmarks → ⋮ (menu) → "Import bookmarks" → Select HTML file
   - Export passwords (if needed):
     chrome://settings/passwords → ⋮ → "Export passwords"
   - Import to TEST-CLEAN profile:
     chrome://settings/passwords → ⋮ → "Import passwords"
   - Expected: Bookmarks and passwords transferred
   - Verify: Spot-check 5 random bookmarks and 3 passwords
   ```

4. **Abandon corrupted profile** — WHY → Eliminate threat vector → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Switch to TEST-CLEAN profile permanently
   - Delete old profile:
     chrome://settings/people → Click old profile → "Remove" button
   - Expected: Old profile data deleted (browser will confirm)
   - Verify: Profile selector no longer shows old profile
   ```

**Evidence Required:**
- [ ] Screenshot of TEST-CLEAN profile with chrome://extensions (empty)
- [ ] Screenshot of affected site in TEST-CLEAN (symptom absent)
- [ ] Bookmark export file saved
- [ ] Confirmation dialog for old profile deletion

**Verification Checklist:**
- [ ] TEST-CLEAN profile symptom-free for 24 hours
- [ ] Bookmarks and passwords accessible
- [ ] No extensions present in TEST-CLEAN
- [ ] Old profile removed from chrome://settings/people

**Escalation Trigger:** If symptom appears in TEST-CLEAN profile within 1 hour → go to Level 2

---

### Level 2: Browser Settings Reset

**Purpose:** Reset policies and settings without losing data

**Verdict Confidence:** 70% — Removes policy-based persistence, preserves user data

**Immediate Actions:**

1. **Backup current state** — WHY → Enable rollback if needed → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Export bookmarks (chrome://bookmarks → Export)
   - Export passwords (chrome://settings/passwords → Export)
   - Screenshot chrome://extensions (with IDs visible)
   - Screenshot chrome://policy
   - List of installed extensions (manual notes)
   - Expected: All data exportable
   - Verify: Open exported files, confirm not empty
   ```

2. **Perform browser reset** — WHY → Remove all policies and extensions → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Navigate to chrome://settings/reset
   - Click "Restore settings to their original defaults"
   - Read confirmation dialog carefully
   - Click "Reset settings" button
   - Wait for reset to complete (may take 30-60 seconds)
   - Expected: Browser restarts automatically
   - Verify: chrome://extensions is empty, chrome://policy is empty
   ```

3. **Verify reset scope** — WHY → Confirm what was preserved vs removed → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Check PRESERVED:
     ✓ Bookmarks (chrome://bookmarks)
     ✓ Passwords (chrome://settings/passwords)
     ✓ History (chrome://history)
     ✓ Autofill data (chrome://settings/addresses)

   - Check REMOVED:
     ✗ All extensions (chrome://extensions)
     ✗ Startup pages (chrome://settings/onStartup)
     ✗ Search engine (chrome://settings/searchEngines)
     ✗ Pinned tabs
     ✗ Content settings (chrome://settings/content)
     ✗ Cookies and site data

   - Expected: Preserved data intact, removed items reset to defaults
   - Verify: Spot-check 3 bookmarks, 2 passwords, startup page is "New Tab"
   ```

4. **Test symptom resolution** — WHY → Confirm threat eliminated → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Navigate to affected site
   - Perform original trigger action
   - Test in normal + Incognito + post-restart
   - Expected: Symptom completely absent
   - Verify: 24-hour monitoring shows no recurrence
   ```

**Evidence Required:**
- [ ] Pre-reset screenshots (extensions, policy, settings)
- [ ] Exported bookmark and password files
- [ ] Post-reset screenshots (empty extensions, empty policy)
- [ ] Symptom test results (absent)

**Verification Checklist:**
- [ ] chrome://extensions empty
- [ ] chrome://policy empty
- [ ] Bookmarks and passwords intact
- [ ] Symptom absent for 24 hours
- [ ] No unexpected extensions reinstalled

**Escalation Trigger:** If symptom reappears within 4 hours OR extensions reinstall automatically → go to Level 3

---

### Level 3: Nuclear Profile Removal

**Purpose:** Complete profile wipe with manual backup

**⚠️ WARNING:** This is a destructive operation. All browser data will be lost except manually backed up items.

**Verdict Confidence:** 95% — Eliminates all browser-level persistence

**Immediate Actions:**

1. **Complete data backup** — WHY → Last chance to preserve data → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Export bookmarks (chrome://bookmarks)
   - Export passwords (chrome://settings/passwords)
   - Screenshot all important tabs (URLs visible)
   - Note installed extensions you want to reinstall
   - Screenshot chrome://settings for preferences
   - Expected: All critical data captured
   - Verify: Review exported files, screenshot clarity
   ```

2. **Close browser completely** — WHY → Unlock profile files → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - File → Exit (or Cmd+Q on macOS)
   - Wait 10 seconds
   - Check Task Manager (Windows) / Activity Monitor (macOS) / htop (Linux)
   - Expected: No chrome/brave/edge processes running
   - Verify: Search for "chrome" in process list → zero results
   ```

3. **Backup profile directory** — WHY → Enable forensic analysis or recovery → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY

   **macOS:**
   ```bash
   # Quit Chrome completely first
   mv ~/Library/Application\ Support/Google/Chrome ~/Desktop/Chrome-backup-$(date +%s)
   # Relaunch Chrome (creates fresh profile)
   ```

   **Windows (PowerShell):**
   ```powershell
   # Close Chrome completely first
   Move-Item "$env:LOCALAPPDATA\Google\Chrome" "$env:USERPROFILE\Desktop\Chrome-backup-$(Get-Date -Format yyyyMMddHHmmss)"
   # Relaunch Chrome
   ```

   **Linux:**
   ```bash
   # Quit Chrome completely first
   mv ~/.config/google-chrome ~/.config/google-chrome.backup-$(date +%s)
   # Relaunch Chrome
   ```

   **Expected Result:** Profile directory moved to Desktop/home directory with timestamp

   **HOW TO VERIFY:**
   - Check Desktop/home for backup folder
   - Backup folder size > 0 bytes (should be 50MB-500MB typically)
   - Original profile directory no longer exists at source location

4. **Relaunch browser** — WHY → Create fresh profile automatically → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Open browser normally
   - Expected: Welcome screen appears (first-run experience)
   - Verify: chrome://extensions empty, chrome://policy empty
   - Test affected site → symptom absent
   ```

5. **Selective data restoration** — WHY → Restore only trusted data → EXACT ACTION → EXPECTED RESULT → HOW TO VERIFY
   ```
   - Import bookmarks (chrome://bookmarks → Import)
   - Import passwords (chrome://settings/passwords → Import)
   - Do NOT restore old profile directory
   - Manually reinstall ONLY trusted extensions from Chrome Web Store
   - Expected: Basic functionality restored, threat eliminated
   - Verify: Test all restored items, symptom still absent
   ```

**Evidence Required:**
- [ ] Screenshot of backup folder on Desktop (with timestamp visible)
- [ ] Screenshot of Chrome welcome screen on first launch
- [ ] Screenshot of empty chrome://extensions after relaunch
- [ ] Symptom test results (absent)

**Verification Checklist:**
- [ ] Backup folder exists and is accessible
- [ ] Fresh profile confirmed (chrome://version shows new profile path)
- [ ] Symptom absent in fresh profile
- [ ] Only trusted extensions installed manually
- [ ] Bookmarks and passwords restored
- [ ] 48-hour monitoring shows no recurrence

**Escalation Trigger:** If symptom appears in fresh profile within 1 hour OR browser profile re-corrupts spontaneously → go to Level 4

---

### Level 4: Seek Professional Help

**Purpose:** Acknowledge system-level compromise beyond browser scope

**🚨 CRITICAL:** The following conditions indicate compromise outside browser control

**Triggers:**
- Symptoms persist across fresh profile AND fresh browser install
- Browser settings revert immediately (within seconds) after changes
- Extensions reinstall automatically within minutes of removal
- System-level certificate injection confirmed (see Certificate Store section)
- Browser uninstallation fails or browser reinstalls itself
- Hosts file modifications persist across cleanups
- DNS settings locked at system level

**Likely Cause:** System-level malware, rootkit, or enterprise MDM

**Immediate Actions:**

1. **Document the scope** — WHY → Provide evidence to security professional
   ```
   - Export ALL evidence collected in previous levels
   - Screenshot chrome://policy with timestamp
   - Screenshot certificate stores (see Certificate Store section)
   - Note timeline of symptoms and remediation attempts
   - Check system-wide proxy settings:
     * macOS: System Preferences → Network → Advanced → Proxies
     * Windows: Settings → Network & Internet → Proxy
     * Linux: System Settings → Network → Network Proxy
   - Document any locked system settings
   ```

2. **Isolate system** — WHY → Prevent further compromise spread
   ```
   - Disconnect from network (Wi-Fi off, ethernet unplugged)
   - Do NOT use system for sensitive activities (banking, passwords)
   - Do NOT attempt further cleanups (may alert malware)
   ```

3. **Seek qualified help** — WHY → Requires expertise beyond browser scope
   ```
   - Corporate environment → Contact IT security team immediately
   - Personal device → Contact professional malware removal service
   - Provide all collected evidence
   - Mention attempted remediation steps to avoid duplication
   ```

4. **Prepare for system wipe** — WHY → Likely outcome for severe infections
   ```
   - Backup personal files to EXTERNAL drive (not cloud sync)
   - Scan backups with updated antivirus before restoring
   - Prepare list of software to reinstall
   - Document critical settings and configurations
   - Change ALL passwords from a CLEAN device after system rebuild
   ```

**Evidence Required:**
- [ ] Complete evidence package from all previous levels
- [ ] System proxy settings screenshot
- [ ] Certificate store screenshots
- [ ] Timeline document (dates/times of symptoms and fixes)
- [ ] List of security tools already attempted

**OUT OF SCOPE:**
This playbook does NOT cover:
- OS-level rootkit removal
- Kernel driver analysis
- Network traffic interception at router level
- Firmware-level persistence
- Hardware keyloggers

**Professional Resources:**
- SANS Internet Storm Center: https://isc.sans.edu/
- Reddit: r/techsupport (verify advice with multiple sources)
- Local cybersecurity firms (Google: "malware removal [your city]")

---

## CROSS-PLATFORM PATH REFERENCES

### Browser Profile Default Paths

| OS | Chrome Profile Default Path |
|----|----------------------------|
| macOS | `~/Library/Application Support/Google/Chrome/Default/` |
| Windows | `%LOCALAPPDATA%\Google\Chrome\User Data\Default\` |
| Linux | `~/.config/google-chrome/Default/` |

### Browser Data Directories (All Variants)

| Browser | macOS Path | Windows Path | Linux Path |
|---------|-----------|--------------|------------|
| Chrome | `~/Library/Application Support/Google/Chrome/` | `%LOCALAPPDATA%\Google\Chrome\` | `~/.config/google-chrome/` |
| Brave | `~/Library/Application Support/BraveSoftware/Brave-Browser/` | `%LOCALAPPDATA%\BraveSoftware\Brave-Browser\` | `~/.config/BraveSoftware/Brave-Browser/` |
| Edge | `~/Library/Application Support/Microsoft Edge/` | `%LOCALAPPDATA%\Microsoft\Edge\` | `~/.config/microsoft-edge/` |
| Vivaldi | `~/Library/Application Support/Vivaldi/` | `%LOCALAPPDATA%\Vivaldi\` | `~/.config/vivaldi/` |
| Chromium | `~/Library/Application Support/Chromium/` | `%LOCALAPPDATA%\Chromium\` | `~/.config/chromium/` |

### Key Files Within Profile

| File | Purpose | Location (relative to profile) |
|------|---------|-------------------------------|
| Preferences | Extension settings, policies | `Default/Preferences` |
| Secure Preferences | Encrypted settings | `Default/Secure Preferences` |
| Extensions | Extension code | `Default/Extensions/[ID]/[version]/` |
| Local Storage | Site storage | `Default/Local Storage/leveldb/` |
| Cookies | Cookie database | `Default/Cookies` |
| History | Browsing history | `Default/History` |
| Login Data | Saved passwords | `Default/Login Data` |

---

## PLATFORM-SPECIFIC DIAGNOSTIC COMMANDS

### Extension Directory Inspection

**macOS:**
```bash
ls -la ~/Library/Application\ Support/Google/Chrome/Default/Extensions/
```

**Windows (PowerShell):**
```powershell
Get-ChildItem "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Extensions\"
```

**Linux:**
```bash
ls -la ~/.config/google-chrome/Default/Extensions/
```

---

### Preferences File Search

**Purpose:** Find policy or enterprise keywords in browser preferences

**macOS:**
```bash
grep -i "policy\|enterprise\|extensioninstall" ~/Library/Application\ Support/Google/Chrome/Default/Preferences
```

**Linux:**
```bash
grep -i "policy\|enterprise\|extensioninstall" ~/.config/google-chrome/Default/Preferences
```

**Windows (PowerShell):**
```powershell
Select-String -Path "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Preferences" -Pattern "policy|enterprise|extensioninstall"
```

---

### Find Recently Modified Extensions

**Purpose:** Identify newly installed or updated extensions

**macOS / Linux:**
```bash
# macOS
find ~/Library/Application\ Support/Google/Chrome/Default/Extensions/ -type d -name "*" -mtime -7

# Linux
find ~/.config/google-chrome/Default/Extensions/ -type d -name "*" -mtime -7
```

**Windows (PowerShell):**
```powershell
Get-ChildItem "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Extensions\" -Recurse | 
Where-Object {$_.LastWriteTime -gt (Get-Date).AddDays(-7)} | 
Select-Object FullName, LastWriteTime
```

---

## CERTIFICATE STORE INSPECTION

### Purpose
Browser-hijacking malware may install rogue root certificates to intercept HTTPS traffic (MITM attacks). These certificates allow malware to decrypt and modify encrypted web traffic.

### macOS Certificate Check

**GUI Method:**
1. Open **Keychain Access.app** (Spotlight → "Keychain Access")
2. Select **System** keychain (left sidebar)
3. Click **Certificates** category
4. Sort by **Date Modified** column
5. Look for certificates added after suspicious activity started
6. Check **Issued By** field for unknown Certificate Authorities

**Suspicious Indicators:**
- Certificate issued by unknown company
- Certificate added recently (within days of symptom onset)
- Certificate name contains: "Proxy", "Filter", "Intercept", "Inspect"
- Issued To = Issued By (self-signed)

**Terminal Method:**
```bash
# List all system certificates
security find-certificate -a -p /Library/Keychains/System.keychain

# Search for specific certificate
security find-certificate -c "Suspicious CA Name" /Library/Keychains/System.keychain

# List certificates added in last 30 days (requires parsing)
security dump-keychain /Library/Keychains/System.keychain | grep -A 10 "cdat"
```

**How to Remove Rogue Certificate:**
```bash
# GUI method (recommended):
# 1. Open Keychain Access
# 2. Select System keychain
# 3. Find suspicious certificate
# 4. Right-click → Delete "[Certificate Name]"
# 5. Enter admin password to confirm

# Terminal method (advanced):
sudo security delete-certificate -c "Suspicious CA Name" /Library/Keychains/System.keychain
```

---

### Windows Certificate Check

**GUI Method:**
1. Win+R → type `certmgr.msc` → Enter
2. Expand **Trusted Root Certification Authorities**
3. Click **Certificates** folder
4. Sort by **Valid From** date or **Issued By** column
5. Look for unfamiliar issuers

**Suspicious Indicators:**
- Issuer name doesn't match known CAs (VeriSign, DigiCert, Let's Encrypt, etc.)
- Valid From date matches symptom onset
- Friendly Name contains: "Proxy", "Web Filter", "Corporate"
- Personal certificates in Trusted Root (red flag)

**PowerShell Method:**
```powershell
# List all root certificates (machine-wide)
Get-ChildItem -Path Cert:\LocalMachine\Root | Format-Table Subject, Issuer, NotBefore, Thumbprint

# List user-specific root certificates
Get-ChildItem -Path Cert:\CurrentUser\Root | Format-Table Subject, Issuer, NotBefore, Thumbprint

# Search for specific issuer
Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object {$_.Issuer -like "*Suspicious*"}

# Check recently added certificates (last 30 days)
Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object {$_.NotBefore -gt (Get-Date).AddDays(-30)}
```

**How to Remove Rogue Certificate:**
```powershell
# Find certificate thumbprint first
Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object {$_.Subject -like "*Suspicious CA*"}

# Remove by thumbprint (replace with actual thumbprint)
Get-ChildItem -Path Cert:\LocalMachine\Root\[THUMBPRINT] | Remove-Item

# Alternative GUI method:
# 1. certmgr.msc
# 2. Trusted Root Certification Authorities → Certificates
# 3. Right-click suspicious cert → Delete
```

---

### Linux Certificate Check

**System Certificates:**
```bash
# List system-wide trusted certificates
ls -la /etc/ssl/certs/
ls -la /usr/share/ca-certificates/

# Check recently modified certificates (last 30 days)
find /etc/ssl/certs/ -type f -mtime -30 -ls
find /usr/share/ca-certificates/ -type f -mtime -30 -ls

# Search for specific certificate
grep -r "Suspicious CA" /etc/ssl/certs/
```

**Chrome/Chromium NSS Database:**
```bash
# List certificates in Chrome's NSS database
certutil -d sql:$HOME/.pki/nssdb -L

# List with details
certutil -d sql:$HOME/.pki/nssdb -L -h all

# View specific certificate
certutil -d sql:$HOME/.pki/nssdb -L -n "Certificate Nickname"

# Search for recently added (check modification date)
ls -lt $HOME/.pki/nssdb/
```

**How to Remove Rogue Certificate:**
```bash
# System certificates (requires root)
# Debian/Ubuntu:
sudo rm /usr/local/share/ca-certificates/suspicious-cert.crt
sudo update-ca-certificates --fresh

# Fedora/RHEL:
sudo rm /etc/pki/ca-trust/source/anchors/suspicious-cert.crt
sudo update-ca-trust

# Arch:
sudo rm /etc/ca-certificates/trust-source/anchors/suspicious-cert.crt
sudo trust extract-compat

# Chrome NSS database:
certutil -d sql:$HOME/.pki/nssdb -D -n "Certificate Nickname"
```

---

### Certificate Validation

**For ANY suspicious certificate, validate legitimacy:**

1. **Check issuer against known CA list:**
   - https://ccadb-public.secure.force.com/mozilla/IncludedCACertificateReport
   - Search for issuer name
   - If not found → likely rogue

2. **Google the certificate name:**
   - Search: "[Certificate Name] root CA"
   - Check if it's a known corporate/school proxy
   - Check if reported as malware

3. **Verify certificate purpose:**
   - Business environment → may be legitimate corporate proxy
   - Home user → almost certainly malicious unless you installed VPN/security software

4. **Cross-reference with timeline:**
   - Certificate added AFTER symptoms started → highly suspicious
   - Certificate exists on clean device → likely legitimate

---

## WINDOWS GROUP POLICY & REGISTRY INSPECTION

### Check Applied Group Policies

**Purpose:** Identify if browser is controlled by Windows domain policy

**PowerShell (run as user):**
```powershell
# Full policy report for current user
gpresult /Scope User /v | Select-String -Pattern "Chrome|Browser|Extension"

# Full policy report including computer policies
gpresult /Scope Computer /v | Select-String -Pattern "Chrome|Browser"

# HTML report for detailed analysis
gpresult /H C:\Users\$env:USERNAME\Desktop\gpresult.html /F
```

**Interpretation:**
- If output contains Chrome policies → browser managed by domain
- Note "Winning GPO" field → identifies controlling policy source
- Check "Applied Group Policy Objects" → list of active policies

---

### Check Registry Policies

**Purpose:** Find browser policies set via Windows Registry

**Check machine-wide policies (all users):**
```powershell
# Chrome policies
reg query "HKLM\SOFTWARE\Policies\Google\Chrome"
reg query "HKLM\SOFTWARE\Policies\Google\Chrome\ExtensionInstallForcelist"

# Chromium policies (affects Brave, Edge, etc.)
reg query "HKLM\SOFTWARE\Policies\Chromium"
```

**Check user-specific policies (current user only):**
```powershell
# Chrome user policies
reg query "HKCU\SOFTWARE\Policies\Google\Chrome"

# Chromium user policies
reg query "HKCU\SOFTWARE\Policies\Chromium"
```

**Export for detailed analysis:**
```powershell
# Export Chrome policies to file
reg export "HKLM\SOFTWARE\Policies\Google\Chrome" C:\Users\$env:USERNAME\Desktop\chrome-policies-machine.reg

reg export "HKCU\SOFTWARE\Policies\Google\Chrome" C:\Users\$env:USERNAME\Desktop\chrome-policies-user.reg

# Open .reg file in notepad to review
notepad C:\Users\$env:USERNAME\Desktop\chrome-policies-machine.reg
```

**Common Malicious Registry Keys:**
- `ExtensionInstallForcelist` → forces extension installation
- `ExtensionInstallBlocklist` → prevents extension removal
- `HomepageLocation` → locks homepage
- `RestoreOnStartupURLs` → forces startup pages
- `ProxyMode` → hijacks proxy settings
- `URLBlocklist`/`URLAllowlist` → controls site access

---

### Remove Registry Policies (Advanced)

**⚠️ WARNING:** Only remove registry keys if you are certain they are malicious. Incorrect deletion can break browser or system.

**Remove machine-wide policy:**
```powershell
# Requires admin rights
# Delete entire Chrome policy key (nuclear option)
reg delete "HKLM\SOFTWARE\Policies\Google\Chrome" /f

# Delete specific subkey (safer)
reg delete "HKLM\SOFTWARE\Policies\Google\Chrome\ExtensionInstallForcelist" /f
```

**Remove user-specific policy:**
```powershell
# Delete user Chrome policies
reg delete "HKCU\SOFTWARE\Policies\Google\Chrome" /f
```

**Verify removal:**
```powershell
# Should return "ERROR: The system was unable to find the specified registry key or value."
reg query "HKLM\SOFTWARE\Policies\Google\Chrome"
```

**Alternative GUI Method:**
```
1. Win+R → regedit → Enter
2. Navigate to: HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Google\Chrome
3. Right-click policy key → Delete
4. Confirm deletion
5. Restart browser
6. Verify: chrome://policy should be empty
```

---

### Check Third-Party Policy Enforcement Tools

**Purpose:** Detect policy management software

```powershell
# Check for common policy management software
Get-Service | Where-Object {$_.DisplayName -like "*Policy*" -or $_.DisplayName -like "*Management*"}

# Check installed programs
Get-WmiObject -Class Win32_Product | Where-Object {$_.Name -like "*Policy*" -or $_.Name -like "*Admin*"}

# Check scheduled tasks (malware may use tasks to re-apply policies)
Get-ScheduledTask | Where-Object {$_.TaskName -like "*Chrome*" -or $_.TaskName -like "*Browser*"}
```

---

## LINUX SYSTEM-WIDE BROWSER POLICIES

### Check System Policy Directories

**Purpose:** Find browser policies set at system level (not user profile)

**Chrome Policy Directories:**
```bash
# Managed policies (cannot be overridden by user)
ls -la /etc/opt/chrome/policies/managed/
cat /etc/opt/chrome/policies/managed/*.json

# Recommended policies (can be overridden by user)
ls -la /etc/opt/chrome/policies/recommended/
cat /etc/opt/chrome/policies/recommended/*.json
```

**Chromium Policy Directories:**
```bash
# Chromium and derivatives (Brave, Vivaldi, etc.)
ls -la /etc/chromium/policies/managed/
cat /etc/chromium/policies/managed/*.json

ls -la /etc/chromium/policies/recommended/
cat /etc/chromium/policies/recommended/*.json
```

**Interpretation:**
- JSON files in these directories apply to ALL users on system
- `managed/` policies cannot be changed in chrome://settings
- `recommended/` policies can be overridden by user

---

### Policy File Inspection

**Example Policy File:**
```bash
cat /etc/opt/chrome/policies/managed/test_policy.json
```

**Common Malicious Policy Contents:**
```json
{
  "ExtensionInstallForcelist": [
    "abcdefghijklmnopqrstuvwxyz123456;https://malicious-site.com/update.xml"
  ],
  "HomepageLocation": "https://hijacked-homepage.com",
  "RestoreOnStartupURLs": [
    "https://unwanted-site.com"
  ]
}
```

---

### Remove System-Level Policies

**⚠️ WARNING:** Requires root/sudo access. Only remove if you control the system.

**Remove policy files:**
```bash
# Backup before removal
sudo cp -r /etc/opt/chrome/policies /tmp/chrome-policies-backup-$(date +%s)

# Remove managed policies
sudo rm /etc/opt/chrome/policies/managed/*.json

# Remove recommended policies
sudo rm /etc/opt/chrome/policies/recommended/*.json

# Verify removal
ls /etc/opt/chrome/policies/managed/
# Should show: ls: cannot access '/etc/opt/chrome/policies/managed/': No such file or directory
# OR: empty directory
```

**Restart browser and verify:**
```bash
# Close all browser instances
pkill chrome

# Reopen browser
google-chrome &

# Check chrome://policy (should be empty)
```

---

### Check Package Installation Source

**Purpose:** Verify browser installed from official repository

**Debian/Ubuntu:**
```bash
# List installed Chrome packages
dpkg -l | grep chrome

# Check package source
apt-cache policy google-chrome-stable

# Output should show:
# 500 http://dl.google.com/linux/chrome/deb/ stable/main
# If different source → potentially compromised package
```

**Fedora/RHEL:**
```bash
# List Chrome packages
rpm -qa | grep chrome

# Check package info
dnf info google-chrome-stable

# Check repository
dnf repolist | grep chrome

# Verify package signature
rpm --checksig $(rpm -q google-chrome-stable)
```

**Arch:**
```bash
# List Chrome packages
pacman -Q | grep chrome

# Check package source
pacman -Si google-chrome

# Check which repository
pacman -Qi google-chrome | grep Repository
```

**Suspicious Signs:**
- Package installed from unknown repository
- Package signature verification fails
- Package version doesn't match official Google releases

---

### Check Browser Launch Scripts

**Purpose:** Detect wrapper scripts that inject policies at launch

```bash
# Find browser binary
which google-chrome

# Check if it's a script or binary
file $(which google-chrome)

# If it's a script, view contents
cat $(which google-chrome)

# Legitimate script should eventually call /opt/google/chrome/chrome
# Suspicious: script sets CHROME_FLAGS environment variables
# Suspicious: script modifies $HOME/.config before launch
```

**Check systemd user services:**
```bash
# List user services
systemctl --user list-units | grep chrome

# Check service file if exists
systemctl --user cat chrome-something.service

# Disable suspicious service
systemctl --user disable chrome-something.service
```

---

## QUICK REFERENCE

### Chrome Internal URLs

| URL | Purpose |
|-----|---------|
| `chrome://extensions` | Manage extensions (enable Developer mode for IDs) |
| `chrome://policy` | View enforced policies (export JSON) |
| `chrome://management` | Check if browser is managed |
| `chrome://settings/syncSetup/advanced` | Sync settings (check what's syncing) |
| `chrome://serviceworker-internals` | Active service workers |
| `chrome://settings/content/all` | Site permissions and storage |
| `chrome://settings/clearBrowserData` | Clear browsing data (time range filters) |
| `chrome://settings/reset` | Reset browser settings |
| `chrome://settings/people` | Manage profiles |
| `chrome://version` | Browser version and profile path |
| `chrome://about` | List all chrome:// URLs |

### Common Diagnostic Shortcuts

| Shortcut | macOS | Windows/Linux | Purpose |
|----------|-------|---------------|---------|
| Incognito | Cmd+Shift+N | Ctrl+Shift+N | Test without extensions |
| DevTools | Cmd+Option+I | F12 or Ctrl+Shift+I | Inspect DOM, console, network |
| View Source | Cmd+Option+U | Ctrl+U | See page HTML |
| Hard Reload | Cmd+Shift+R | Ctrl+Shift+R | Bypass cache |
| Task Manager | Search "Task Manager" in menu | Shift+Esc | Kill processes/extensions |
| Downloads | Cmd+Shift+J | Ctrl+J | Check downloaded files |

### Evidence Collection Quick Commands

**Export Policy:**
```
chrome://policy → "Export policies as JSON" button
```

**Screenshot Extensions with IDs:**
```
chrome://extensions → Developer mode ON → Screenshot
```

**Dump Service Workers:**
```
chrome://serviceworker-internals → Copy table for affected origin
```

**DOM Inspection:**
```javascript
// In DevTools Console (F12)
console.log(document.documentElement.outerHTML.substring(0, 5000));
```

**Profile Path:**
```
chrome://version → "Profile Path" row → copy full path
```

---

## ADVANCED TECHNIQUES

### Extension Forensics

<details>
<summary>Click to expand: Deep extension analysis</summary>

**Extract Extension Source Code:**

**macOS:**
```bash
# Navigate to extension directory
cd ~/Library/Application\ Support/Google/Chrome/Default/Extensions/[EXTENSION_ID]/[VERSION]/

# List files
ls -la

# Read manifest
cat manifest.json | jq .

# Search for suspicious patterns
grep -r "eval\|atob\|fromCharCode" .
grep -r "http\|https" . | grep -v "chrome-extension"
```

**Windows (PowerShell):**
```powershell
# Navigate to extension
cd "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Extensions\[EXTENSION_ID]\[VERSION]"

# List files
Get-ChildItem -Recurse

# Read manifest
Get-Content manifest.json | ConvertFrom-Json

# Search for suspicious code
Select-String -Path *.js -Pattern "eval|atob|fromCharCode"
```

**Linux:**
```bash
# Navigate to extension
cd ~/.config/google-chrome/Default/Extensions/[EXTENSION_ID]/[VERSION]/

# Find all JavaScript files
find . -name "*.js" -type f

# Search for obfuscation
grep -r "eval\|atob" .

# Check external connections
grep -r "http" . | grep -v "chrome-extension"
```

**Suspicious Code Patterns:**
- `eval()` — code execution from strings
- `atob()` / `btoa()` — base64 encoding/decoding (obfuscation)
- `String.fromCharCode()` — character code obfuscation
- `document.write()` — DOM injection
- External script loading from non-HTTPS sources
- Obfuscated variable names (single letters, random strings)
- Large blocks of encoded data

</details>

---

### Network Traffic Analysis

<details>
<summary>Click to expand: Capture and analyze browser traffic</summary>

**Use DevTools Network Tab:**
```
1. F12 → Network tab
2. Check "Preserve log" checkbox
3. Reproduce symptom
4. Filter by:
   - XHR/Fetch (API calls)
   - JS (script loads)
   - Doc (page navigations)
5. Look for:
   - Unexpected domains
   - Redirect chains (3xx status codes)
   - Failed requests (4xx, 5xx)
   - Large data uploads (POST requests)
```

**Export HAR File (HTTP Archive):**
```
1. DevTools → Network tab
2. Right-click on request list → "Save all as HAR with content"
3. Upload to: https://toolbox.googleapps.com/apps/har_analyzer/
4. Analyze timing, requests, domains
```

**Check for DNS Leaks:**
```
1. Navigate to: https://www.dnsleaktest.com/
2. Click "Extended test"
3. Note DNS servers used
4. If unexpected servers → proxy/VPN hijacking
```

</details>

---

### Browser Launch Parameter Injection

<details>
<summary>Click to expand: Detect command-line hijacking</summary>

**Check Browser Shortcut Parameters (Windows):**
```
1. Right-click Chrome desktop shortcut → Properties
2. Check "Target" field:
   - Should be: "C:\Program Files\Google\Chrome\Application\chrome.exe"
   - Suspicious: Extra parameters like --proxy-server, --load-extension
3. Check "Start in" field (should be blank or Chrome install dir)
```

**macOS Launch Agents:**
```bash
# Check user launch agents
ls -la ~/Library/LaunchAgents/ | grep -i chrome

# Check system launch agents
ls -la /Library/LaunchAgents/ | grep -i chrome

# View suspicious agent
cat ~/Library/LaunchAgents/com.suspicious.chrome.plist

# Disable agent
launchctl unload ~/Library/LaunchAgents/com.suspicious.chrome.plist
```

**Linux Desktop Files:**
```bash
# Check user desktop entries
grep -r "google-chrome" ~/.local/share/applications/

# Check system desktop entries
grep -r "google-chrome" /usr/share/applications/

# View desktop file
cat ~/.local/share/applications/google-chrome.desktop

# Look for Exec= line with suspicious flags
```

</details>

---

## RESPONSE FORMAT TEMPLATE

Every investigation should document findings in this format:

```markdown
## [SYMPTOM NAME] Investigation Report

**Date:** [YYYY-MM-DD]
**Browser:** [Chrome/Brave/Edge] [Version]
**OS:** [macOS/Windows/Linux] [Version]

**Verdict:** [Extension-based / Policy-based / Site-level / System-level] + [Confidence %]

**Root Cause:**
[Exact description of threat vector]

**Evidence:**
- Screenshot: chrome://extensions → [link]
- Screenshot: chrome://policy → [link]
- Extension ID: [alphanumeric ID]
- crxcavator.io report: [link]

**Immediate Actions Taken:**
1. [Action] → [Result]
2. [Action] → [Result]
3. [Action] → [Result]

**Verification Results:**
- [ ] Normal window test — PASS/FAIL
- [ ] Incognito test — PASS/FAIL
- [ ] Post-restart confirmation — PASS/FAIL
- [ ] 24-hour monitoring — PASS/FAIL

**Escalation Level:** [1/2/3/4] or [None]

**Follow-up Required:**
- [ ] Change passwords on clean device
- [ ] Audit synced Chrome data
- [ ] Monitor for reinfection (7 days)

**Lessons Learned:**
[What could have prevented this / early warning signs missed]
```

---

## CHANGELOG

**v1.0 (2026-01-26):**
- Initial release
- Cross-platform support (macOS, Windows, Linux)
- 4-level escalation framework
- Threat intelligence integration (crxcavator.io)
- Sync hijacking detection
- Certificate store inspection
- Windows Group Policy checks
- Linux system policy checks
- Evidence collection standardization

---

## CONTRIBUTING

Improvements welcome. Submit additions that meet these criteria:
- Command-first approach (terminal over GUI)
- Proof-based verification required
- Cross-platform coverage (all 3 OSes)
- No speculation — only documented techniques
- Include evidence collection method

**Out of scope:**
- OS-level malware removal
- Network stack manipulation
- Hardware forensics
- Generic security advice

---

**END OF PLAYBOOK**

*For professional security consultation, this playbook should be paired with system-level analysis when Level 4 escalation is triggered.*
