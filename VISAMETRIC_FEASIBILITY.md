# VisaMetric Germany/Ireland Appointment Monitor - Feasibility Assessment

## 1. System Architecture

### URLs
- **Info page**: `https://www.visametric.com/Ireland/Germany/en`
- **Booking system**: `https://ie-appointment.visametric.com/en` (separate subdomain)
- **Appointment form**: `https://ie-appointment.visametric.com/en/appointment-form`

### Tech Stack
- **Backend**: Laravel (PHP) - evidenced by CSRF tokens (`_token`), XSRF-TOKEN cookies, Blade templates
- **Frontend**: jQuery 3.5.1, Bootstrap 4 Alpha, SweetAlert2, jQuery UI Datepicker
- **CDN/Security**: Cloudflare (CF-Ray headers, 403 on non-browser requests)
- **Analytics**: Google Analytics (G-WKB4918H9T)
- **No SPA/Angular** - traditional server-rendered pages with jQuery AJAX

### API Endpoints (All POST, all require CSRF token + session cookies)
| Endpoint | Purpose | Parameters |
|---|---|---|
| `/en/appointment-form` | Entry (form POST after CAPTCHA) | `_token`, `mailConfirmCode` (CAPTCHA answer) |
| `/en/getvisitingcountry` | Get countries for visa type | `getvisaapptypeid`, `currentlang` |
| `/en/getvisacity` | Get cities for country | `getvisaapptypeid`, `currentlang` |
| `/en/getvisaoffice` | Get offices for city | `getvisacityid`, `getvisacountryid` |
| `/en/getvisaofficetype` | Get service types for office | `getvisaofficeid`, `getvisacityid`, `getvisacountryid` |
| `/en/getcalendarstatus` | Get calendar type | `getvisaofficeid`, `getservicetypeid`, `getvisacountryid` |
| **`/en/getavailablefirstdate`** | **CHECK SLOT AVAILABILITY** | `serviceType`, `totalPerson`, `getOfficeID`, `calendarType`, `getConsular` |
| `/en/getdate` | Get available dates for calendar | `consularid`, `exitid`, `servicetypeid`, `calendarType`, `totalperson` |
| `/en/senddate` | Get time slots for a date | `fulldate`, `totalperson`, various IDs + `personalinfo` (encrypted) |
| `/en/personal/passport-control` | Check passport not already booked | `passport[]`, `country_id` |
| `/en/personal/email-control` | Check email not already booked | `email[]`, `country_id` |
| `/en/personal/phone-control` | Check phone not already booked | `phone[]`, `country_id` |
| `/en/confirmCodeSendMail` | Email verification step | `confirmCode`, `emailValControl` (encrypted) |
| `/en/controldate` | Validate selected date/time | `dateall`, `personCountTotal` |
| `/en/jky45fgd` | Obfuscated endpoint (email check trigger) | `emailCheck`, `personalinfo` (encrypted) |

### Key Response Format
`getavailablefirstdate` returns JSON:
```json
{
  "firstAvailableDate": "<HTML with dates or 'no available date'>",
  "dataDate": "<HTML with first available dates>",
  "isAvailable": true/false
}
```

## 2. Anti-Bot Protections

### Entry Gate: Simple Image CAPTCHA
- **Type**: Server-generated PNG image with 4-digit number (base64 encoded inline)
- **NOT Google reCAPTCHA** - the code has reCAPTCHA commented out! They switched to a simpler CAPTCHA
- **Solving**: Trivial with any OCR (Tesseract, or a simple CAPTCHA solving service). The digits are clear, minimal distortion
- **When**: Only on the initial page (`ie-appointment.visametric.com/en`), not on subsequent API calls

### Cloudflare
- **403 on all curl/httpx requests** - standard Cloudflare browser verification
- However, once inside a browser session, all AJAX calls work freely via `fetch()`
- No Turnstile challenge observed (unlike VFS Global)

### Session Management
- CSRF token required for all POST requests
- XSRF-TOKEN cookie (Laravel standard)
- **20-minute session timer** on the appointment form
- Session established after CAPTCHA solve

### Email Verification
- During booking (not monitoring), applicant email receives a 4-digit code
- Code has a timeout (~2 minutes based on timer.start/timer.reset calls)

### Rate Limiting
- No obvious rate limiting detected on API calls
- All AJAX calls use `async: false` (synchronous), suggesting the devs weren't worried about concurrent abuse

## 3. Booking Flow (6 Steps)

```
1. CAPTCHA Gate      → Solve 4-digit image CAPTCHA → POST to /appointment-form
2. Trip Information  → Select: Application type → Country → City → Office → Service Type → Number of applicants
                       (Each dropdown triggers AJAX cascade)
                       → getavailablefirstdate shows "no dates" or first available dates
3. Personal Info     → Name, surname, nationality, DOB, passport, email, phone
                       → Passport/email/phone duplicate checks via AJAX
4. Preview           → Review all info
5. Contract          → Accept terms, sales contract checkbox
                       → Calendar page shows available dates (datepicker)
                       → Select date → senddate returns time slots
                       → Select time slot
6. Additional Services → Optional paid services (Prime Time, etc.)
                       → Email verification code sent
                       → Enter code → Submit → Payment (credit card online)
```

### The Critical "Slot Check" Happens At:
- **Step 2**: `getavailablefirstdate` - tells you if ANY dates exist (binary yes/no + first dates)
- **Step 5**: `getdate` - returns all available dates for the datepicker calendar
- **Step 5**: `senddate` - returns available time slots for a specific date

## 4. Current Availability (Observed 2026-03-26)

| Category | Available? | First Dates |
|---|---|---|
| Schengen - Tourism/Family&Friend Visit/Transit/Other | **NO** | "There is no available date" |
| National - Working Holiday/Research/Guest Scientist | **YES** | 15-04-2026, 20-04-2026, 27-04-2026 |
| Schengen - EU/EEA and Swiss National Family | Not checked | - |
| Schengen - Business Trips | Not checked | - |

## 5. Comparison to VFS Global

| Aspect | VFS Global | VisaMetric |
|---|---|---|
| **CAPTCHA** | Cloudflare Turnstile (hard) | Simple 4-digit image (trivial) |
| **Login** | Full account login required | No login - CAPTCHA only |
| **Cloudflare** | Aggressive (blocks everything) | Standard (blocks curl, allows browser JS) |
| **API calls from browser** | Work via Angular XHR | Work via jQuery AJAX |
| **API calls from curl** | Blocked (403) | Blocked (403) |
| **Session cost** | ~13MB per browser restart | ~2-3MB per browser restart |
| **Slot check method** | Dropdown re-click triggers API | POST to `getavailablefirstdate` |
| **Check bandwidth** | ~2-5KB per check | ~1-2KB per check (smaller responses) |
| **Session duration** | Unlimited (stay on page) | **20-minute timer** (must re-enter) |
| **Anti-bot sophistication** | High | Low-Medium |
| **Rate limiting** | Unknown/suspected | Not observed |
| **Framework** | Angular SPA | jQuery + Laravel server-rendered |

### Verdict: VisaMetric is SIGNIFICANTLY EASIER than VFS Global

## 6. Recommended Approach

### Option A: Browser-Based Monitor (Recommended)
**Similar to current VFS approach but simpler.**

1. **Launch Playwright/Puppeteer browser** (headless)
2. **Navigate to `ie-appointment.visametric.com/en`**
3. **Solve CAPTCHA**:
   - Screenshot/extract the base64 CAPTCHA image
   - OCR with Tesseract (free, local) - digits are clear
   - Or use CapSolver/2Captcha (cheap, ~$0.001 per solve)
4. **Submit CAPTCHA** → lands on appointment form
5. **Call `getavailablefirstdate` via page.evaluate()** with the right params:
   ```javascript
   fetch('/en/getavailablefirstdate', {
     method: 'POST',
     headers: {'X-CSRF-TOKEN': csrfToken, 'Content-Type': 'application/x-www-form-urlencoded'},
     body: 'serviceType=1&totalPerson=1&getOfficeID=1&calendarType=2&getConsular=1'
   })
   ```
6. **Parse response**: Check `isAvailable` field
7. **If available**: Send Telegram notification with first available dates
8. **Repeat every N seconds** (within the 20-min session window)
9. **Re-do CAPTCHA** every ~18 minutes to refresh session

### Option B: Hybrid Approach (More Efficient)
**Use browser only for session establishment, then direct API calls.**

1. Browser solves CAPTCHA and gets session cookies + CSRF token
2. Extract cookies/CSRF from the browser
3. Use those cookies with `httpx` or `requests` to make direct API calls
4. This avoids the browser overhead for repeated checks
5. Risk: Cloudflare may fingerprint the session differently

### Option C: Pure API (Risky)
Use `curl_cffi` or `cloudscraper` to bypass Cloudflare and make direct API calls without a browser. Higher chance of detection, may work given VisaMetric's weaker protections.

## 7. Effort Estimate

| Task | Estimated Time |
|---|---|
| Basic monitor (browser-based, CAPTCHA OCR) | 3-4 hours |
| Telegram notifications | 30 min (reuse from VFS bot) |
| CAPTCHA solving integration | 1 hour |
| Session refresh logic (20-min cycle) | 1 hour |
| Docker deployment | 30 min (reuse from VFS) |
| **Total** | **~6-7 hours** |

### Key Parameters for Schengen Tourism Monitoring
```
Application type (country): 1  (Schengen - Tourism/Family&Friend Visit)
Visiting country: 1  (Germany)
City: 6  (Dublin)
Office: 1  (Dublin)
Service type (officetype): 1  (NORMAL)
Calendar type: 2  (from getcalendarstatus)
Total person: 1
```

## 8. Cost Estimate (Monthly)

| Item | Cost |
|---|---|
| Hetzner VPS | Already running ($0) |
| Proxy bandwidth | ~200MB/month at 1 check/minute (~$0.50) |
| CAPTCHA solving | ~1300 solves/month (18-min refresh) = ~$1.30 |
| **Total** | **~$2/month** |

Note: Proxy may not even be needed if Cloudflare doesn't IP-block. The CAPTCHA is the main gate, not IP reputation.
