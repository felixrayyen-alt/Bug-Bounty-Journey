# Bug-Bounty-Journey
This is my journey of bug hunting in BugCrowd Especially. Here we are at the first stage until the end.

## Overview

Dokumentasi perjalanan bug hunting Felix di Bugcrowd, mencakup dua target: MATLAB Online (paid bounty) dan Bendigo Bank (VDP). Dimulai dari nol tanpa pengalaman sebelumnya, fokus pada unauthenticated testing dan web application security.

## Target 1: MATLAB Online (Bugcrowd — Paid Bounty)

**Program:** MathWorks / MATLAB Online

**Tipe:** Paid Bug Bounty

**Status:** ❌ Ditinggal

### Yang dilakukan

- Testing XSS di semua input field yang tersedia — semua field terbukti aman, output di-encode dengan benar
- Menemukan login issue saat testing, menghubungi MathWorks support untuk klarifikasi akses
- Semua search engine (3) ke-block saat mencoba recon lebih lanjut
- Tidak ada balasan lanjutan dari MathWorks support

### Kesimpulan

Target ditinggal karena akses terbatas dan tidak ada respons dari support. Attack surface yang bisa dieksplor habis dengan constraint yang ada.

## Target 2: Bendigo Bank VDP (Bugcrowd)

**Program:** Bendigo Bank Responsible Disclosure Program

**Tipe:** VDP (Vulnerability Disclosure Program) — no monetary reward, acknowledgment only

**Status:** 🔄 Ongoing

**Constraint utama:** Unauthenticated testing only, production-safe, max 6 req/sec, no stored XSS, no brute-force pada form submission

**Scope:** `*.bendigobank.com.au`, `*.adelaidebank.com.au`, `*.up.com.au`, dan beberapa domain lain termasuk mobile apps iOS/Android

---

### Phase 1: Initial Recon & XSS Testing

**Target:** `www.bendigobank.com.au`

- Testing SSTI di search bar dengan payload `test{{7*7}}test` — response kosong (`linksList: []`), bukan indikasi SSTI, hanya autocomplete API yang tidak menemukan match
- Testing template injection lain — semua dead end, endpoint adalah pure JSON API tanpa template rendering
- Mencari form input lain menggunakan Google dork: `site:bendigobank.com.au inurl:contact/form/calculator`

---

### Phase 2: Form Submission Analysis

Menemukan tiga public-facing form:

- Home Loan Enquiry (`/personal/home-loans/`)
- Complaint Form (`/contact-us/make-a-complaint/`)
- Customer Compliment Form (`/contact-us/customer-compliments/`)

Submit masing-masing form sekali dengan data dummy, capture response di Burp Suite.

---

### Phase 3: Finding — Internal CRM Headers Exposed

**Vulnerability:** Internal CRM metadata dan backend identifier ter-expose di HTTP response headers pada semua form submission endpoint.

**Headers yang bocor:**

| Header | Deskripsi |
| --- | --- |
| `Bendigosubmissionid` | Internal submission record ID (MongoDB-style) |
| `Bendigoopportunityid` | Salesforce CRM Opportunity ID |
| `Bendigocontactid` | Salesforce CRM Contact ID (linked ke PII user) |
| `Bendigosrnumber` | Internal Service Request number |
| `Bendigofeedbacksource` | CRM routing/source classification |
| `Bendigoisanonymoussubmission` | Internal workflow flag |
| `Bendigoisneedcontact` | Internal workflow flag |
| `Bendigofeedbacktype` | Internal feedback category |

**Tech stack yang teridentifikasi:** Optimizely/Episerver (CMS) + Salesforce (CRM)

**Evidence:** 3 form, 3 response berbeda, semua return header yang sama → systemic issue

**Submission ke Bugcrowd:**

- VRT Category: Sensitive Data Exposure > Disclosure of Secrets > PII Leakage/Exposure
- Severity yang di-set: P3
- **Hasil: Informational (P5)** — reproducible tapi dianggap best-practice gap, tidak ada demonstrated exploitability langsung
- Catatan Bugcrowd: disarankan chain dengan finding lain untuk escalate severity

---

### Phase 4: JavaScript File Analysis

**Tujuan:** Mencari endpoint internal atau referensi ke CRM ID di JS files

**Method:** Enumerate semua JS file yang ke-load di halaman via `performance.getEntriesByType('resource')`, kemudian grep keyword: `contactId`, `submissionId`, `opportunityId`, `srNumber`, `/api/`, `Bendigo`

**Hasil:**

- `/api/` → hit di `app-xxx.min.js` tapi semua logic server-side, tidak ada endpoint yang ter-expose
- `Bendigo` → hit di `customvalidators-xxx.js` — ternyata hanya client-side form validators (phone, email, postcode, recaptcha)
- Semua keyword lain → kosong

**Kesimpulan:** Dead end. Backend logic tidak di-expose ke client.

---

### Phase 5: Subdomain Enumeration

**Method:** `crt.sh/?q=%.bendigobank.com.au` — mendapat 1700+ subdomain dari certificate transparency logs

**Yang hidup (dari Priority 1):**

| Subdomain | Status | Keterangan |
| --- | --- | --- |
| `bpmu.bendigobank.com.au` | ✅ Hidup | BPM UAT — login wall (employee/partner only), skip |
| `demo.bendigobank.com.au` | ✅ Hidup | Interactive e-banking demo, dieksplor lebih lanjut |
| `bpmd/bpmq/bpmp.bendigobank.com.au` | ❌ Forbidden | Access denied |
| `development.assets.app.bendigobank.com.au` | ❌ Forbidden | Access denied |
| `saaslog.test.bendigobank.com.au` | ❌ Forbidden | Access denied |

**CDR API endpoints:** `api.cdr.bendigobank.com.au` — semua path missing/no route

---

### Phase 6: Demo Environment Testing

**Target:** `demo.bendigobank.com.au`

**Tech stack:** Ruby on Rails (`X-Runtime` header), Via Google infrastructure (bukan Cloudflare)

**CSP header** menyebut `env%3Aproduction` → demo environment logging ke DataDog sebagai production

**Yang ditemukan:**

- 5 demo account dengan MD5-hashed account IDs
- Demo user: `BEN104252235` (dari decode cookie `yuulong_fake_session`)
- `customerNumber: 160215` (sequential integer)
- Account group ID, full card IDs, partial card numbers

**IDOR Testing:**

| Test | Hasil |
| --- | --- |
| Unauthenticated access ke `/banking/api/accounts` | 401 — properly protected |
| customerNumber enumeration (160214, 160216) | 404 — endpoint tidak exist |
| Direct account access tanpa auth | 401 — properly protected |
| Transaction ID manipulation | Ignored — server return data sama |

**Kesimpulan:** Demo environment auth solid, tidak ada IDOR yang bisa didemonstrasikan.

---

### Phase 7: CDR API & Up Bank

**CDR API (`api.cdr.bendigobank.com.au`):**

- Semua standard CDR public endpoint (products, discovery/status, well-known) → missing/no route
- Dead end

**Up Bank API (`api.up.com.au`):**

- `/.well-known/openid-configuration` → 200, return full OIDC config (public by design)
- `/api/v1/util/ping` → 401 — butuh auth token
- `/keys.jwks` → 401 — tidak public
- Semua endpoint butuh personal access token
- Constraint unauthenticated only → tidak bisa lanjut tanpa klarifikasi rules

## Lessons Learned

- **"As an attacker I could..."** — setiap finding harus bisa menjawab kalimat ini dengan sesuatu yang konkret dan demonstrable. Tanpa itu, finding kemungkinan besar jatuh di P4-P5
- **Fingerprinting bukan endpoint** — tau tech stack (Salesforce, Optimizely) itu useful tapi bukan finding sendiri. Nilainya ada di chaining ke vuln lain
- **VDP vs Paid Bounty** — VDP hanya kasih acknowledgment, bukan uang. Worth buat belajar dan portfolio, bukan monetisasi
- **Unauthenticated constraint** sangat membatasi attack surface di target perbankan. IDOR yang meaningful hampir selalu butuh minimal dua akun
- **Demo environment** bisa jadi attack surface yang menarik — tech stack beda, kadang security lebih longgar — tapi Bendigo demo-nya solid
- **Subdomain enum** lebih berbuah dari JS hunting untuk target yang backend-nya tertutup
- **Rules of engagement wajib dibaca sebelum testing** — Bendigo eksplisit larang stored XSS, brute-force form, dan credentialed testing

## Tools Used

- **Burp Suite** — intercept, replay, header analysis
- **Browser DevTools (Console)** — JS file enumeration, cookie decode, resource listing
- **crt.sh** — subdomain enumeration via certificate transparency logs
- **Google Dorks** — menemukan form dan endpoint tersembunyi
- **Bugcrowd Platform** — submission dan tracking

## Status

| Target | Status | Hasil |
| --- | --- | --- |
| MATLAB Online | ❌ Ditinggal | Akses terbatas, no response dari support |
| Bendigo Bank — Main Site Forms | ✅ Submitted | P5 Informational — reproducible tapi low impact |
| Bendigo Bank — Demo Environment | ✅ Dieksplor | Auth solid, tidak ada IDOR |
| Bendigo Bank — CDR API | ✅ Dieksplor | Dead end, semua endpoint missing |
| Up Bank — API | ✅ Dieksplor | Semua endpoint butuh auth |

**Next steps yang perlu dipelajari:**

- Pelajari modul BCU: *How to Make Good Submissions* dan *Access Controls*
- Pivot ke target paid bounty baru dengan attack surface yang lebih ramah untuk unauthenticated testing
- Atau eksplor Up Bank lebih lanjut jika rules of engagement mengizinkan credentialed testing dengan developer token
