# Ilm-ul-Quran USA — Web Platform UI Specification

**Project:** WordPress → Next.js rebuild
**Organization:** AL HASANAH FOUNDATION (dba Ilm-ul-Quran USA) · 501(c)(3) · EIN 93-3902300
**Domain:** ilmulquranus.org
**Audience:** Muslim families in the United States and Canada
**Document owner:** Muhammad Nurul Ahsan — Full-Stack Developer & Digital Operations Lead
**For:** UI/UX Designer
**Version:** 2.0 · August 2026 — includes Zoom-automated attendance (supersedes IQU SMS v1.0)

---

## How to read this document

| Section | What it gives you |
|---|---|
| 1 | Why we are rebuilding, and what success looks like |
| 2 | What exists on the current site today — a full audit |
| 3 | What comparable institutions do, and what we should borrow |
| 4 | Three visual directions to choose from, and our recommendation |
| 5 | Design system foundations — tokens, type, grid, motion |
| 6 | Global components that appear on many screens |
| 7 | **Page-by-page specification** — the main body of work |
| 7C | **Attendance & verification engine** — how class attendance is produced automatically |
| 8 | Content we still need to write or shoot |
| 9 | Accessibility, localisation, performance |
| 10 | Handoff checklist and deliverable list |

Every screen in Section 7 follows the same structure: **Purpose → Sections in order → Content per section → Layout → Components → States → Responsive → Notes.** Where a section is marked **NEW**, no equivalent exists on the current site.

---

## 1. Project context

### 1.1 What Ilm-ul-Quran USA is

A nonprofit online Quran and Islamic education platform. Live, teacher-led classes delivered from Bangladesh-based Huffaz, Ulama and Mufti to students in North America. Roughly 250 students, 25+ instructors, 5+ management staff. Nearly half of students study free; the rest pay heavily subsidised fees. Revenue is a mix of tuition and donations (Zeffy), and there is a Zakat scholarship scheme.

This is the single most important framing for the designer: **this is not a commercial academy that happens to be Islamic. It is a charity that teaches.** The site has to work simultaneously as an enrolment funnel for parents and a donation funnel for supporters, without either one undermining the other's credibility.

### 1.2 Why we are moving off WordPress

The current site is WordPress + Twenty Twenty-Four child theme + Gutenberg + a stack of plugins (AirLift, W3 Total Cache, Fluent Forms, Rank Math, a custom `IQU Registration System` plugin at v2.9.9, a custom `Ilm-ul-Quran Companion` block plugin). Layout is trapped inside page content; every visual change is a Gutenberg edit. Caching and hook-timing conflicts have already produced real bugs. There is no student portal at all — enrolment ends in a database row and a Telegram message.

The rebuild targets a Next.js application with a headless data layer, giving us: real component reuse, a proper authenticated portal, faster Core Web Vitals, and layout that lives in code rather than in `post_content`.

### 1.3 Goals for the new UI

1. **Convert browsing parents into enrolled students.** The current enrolment path is a shortcode dropped on a page. It should become a designed, staged flow with visible progress and reassurance at every step.
2. **Convert visitors into donors.** Donation needs its own narrative — where money goes, who it reaches, what a given amount buys.
3. **Make quality visible.** Twenty qualified instructors with real credentials is our strongest asset and it is currently presented as a wall of names with grey placeholder photos.
4. **Give families a home after they enrol.** Schedule, attendance, Sabaq/Sabqi/Manzil progress, reports, invoices — none of this exists on the web today.
5. **Look like an institution, not a template.** Parents are handing us their children's religious education. The visual language has to earn that.
6. **Make every class verifiable.** Today a teacher reports attendance by hand and nobody can confirm any of it. In the new system attendance is produced by Zoom and nobody marks it at all. This is the single largest functional change in the platform and it reshapes the teacher, parent and admin screens — see Section 7D.

### 1.4 Non-goals for v1

- Public-facing course marketplace with self-serve payment for every course (fees are negotiated and subsidised; a rigid checkout would misrepresent us).
- Native mobile apps.
- Live video built in-house. Classes run on Zoom, hosted from the organization's own account; the portal issues the join link and consumes Zoom's events. A custom video stack was evaluated and rejected — it costs roughly twenty times more per month and delays the attendance work by months.
- A public forum or social feed.

---

## 2. Audit of the current site

Pulled directly from the production WordPress install on 30 August 2026.

### 2.1 Page inventory

| Page | Slug | Status | Notes |
|---|---|---|---|
| Home | `/home` | Published | Front page. ~64 KB of block markup. |
| Courses | `/courses` | Published | Tabbed catalog via `easy-tabs-block`. |
| Instructors | `/instructors` | Published | 20 instructor entries. |
| About | `/about` | Published | History, Goal, Org Information, teachers, process. |
| Donate | `/donate` | Published | Stats, subsidy explanation, Fluent Form. |
| Contact | `/contact` | Published | Email/Visit/Text/Follow cards + Fluent Form. |
| Enroll for Free | `/enroll-for-free` | Published | Hosts `[iqu_registration_form]`. |
| Summer Program | `/summer-program` | Published | Hosts `[iqu_summer_form]`. |
| Summer Ilm Camp Results 2026 | `/summer-program-results` | Published | Results leaderboard. |
| enroll | `/enroll` | Published | **Newsletter signup only — misleading URL.** |
| Blog | `/blog` | **Draft** | 67 KB of content, never launched. |
| Student Registration | `/student-registration` | Published | **36 bytes. Empty.** |
| Instructor Registration | `/instructor-registration` | Published | **36 bytes. Empty.** |
| Dashboard | `/dashboard` | Published | **0 bytes. Empty.** |
| Cart / Checkout | `/cart`, `/checkout` | Published | **0 bytes. Orphaned e-commerce stubs.** |
| Privacy Policy | `/privacy-policy` | Published | |
| Terms and Conditions | `/terms-and-conditions` | Published | |

### 2.2 Content types

**Courses** (`ilm_ul_quran_course`) — 6 published:

| Course | Duration | Lessons | Language | Certificate |
|---|---|---|---|---|
| Noorani Qaidah | 2 weeks | 10 | English, Arabic | Yes |
| Quran Nazirah | No limit | No limit | English, Bengali, Arabic, Urdu | Yes |
| Quran Memorization (Hifz) | As required | As required | English, Bengali, Arabic, Urdu | Yes |
| Quranic Arabic Mastery – Level 1 | 1 year | 4/week | English, Bengali | Yes |
| Weekend Islamic Education Program | No limit | No limit | English | No |
| Summer Islamic Education Program | 2.5 months | 45 | English | No |

Each course carries meta fields: `students`, `duration`, `lessons`, `certification`, `support`, `retake`, `language`, `instructors`. Every course page uses the same four-tab layout: Overview / Curriculum & Method / Instructor / Review.

**Instructors** (`iq_instructor`) — 20 published. Fields are effectively name + job title only.

### 2.3 Data layer already in place

| Table | Rows | What it holds |
|---|---|---|
| `iqu_registrations` | 81 | 50 columns. Student details, guardian details, level, schedule preference, language, device, referral, fee preference, coupon, Zakat declaration, payment status. |
| `iqu_zeffy_payments` | 63 | Full Zeffy donation sync — amount, campaign, fund, recurring status, card brand, receipt URL. |
| `iqu_coupons` | 3 | Code, occasion, course type, discount type/value, expiry, max uses. |

Roles registered: `administrator`, `editor`, `author`, `contributor`, `subscriber`, `instructor`, `tutor_instructor`, `css_js_designer`. Only 3 admins and 1 tutor account actually exist — **the roles are declared but the portal was never built.**

### 2.4 What is genuinely working

- The registration form captures a rich, well-considered data set. That schema is the backbone of the portal and should not be thrown away.
- Zeffy donation sync is complete and live. Donation data can drive a real public impact dashboard.
- Course meta fields are consistent enough to render structured course cards.
- Rank Math schema for Course and Organization is already configured.

### 2.5 Gaps — ordered by cost to the business

| # | Gap | Consequence |
|---|---|---|
| 0 | **Attendance is self-reported and unverifiable.** Teachers report by hand; admin has no way to confirm a class happened, that the teacher showed up, or that the student joined. | The organization pays for and reports on teaching it cannot evidence. |
| 1 | **No student/parent portal.** Dashboard page is 0 bytes. | Families have no place to go after enrolling. Everything runs through WhatsApp and Telegram. |
| 2 | **Enrolment is a single long form.** | High abandonment on a 50-field form with no progress indication. |
| 3 | **Instructor pages are name + title.** | Our single strongest trust signal is invisible. No bio, credential, sanad, specialisation, sample recitation, or language. |
| 4 | **No pricing anywhere.** | Parents cannot self-qualify. Every price question becomes a support conversation. |
| 5 | **No testimonials with attribution.** | Home has a testimonial block; About and Courses do not. No photos, no locations, no outcomes. |
| 6 | **Blog is a draft.** | Zero organic content surface despite Rank Math being fully configured. |
| 7 | **No FAQ page.** | Same questions answered manually every day. |
| 8 | **Donate page mixes numbers without a story.** | "250 students / 25+ teachers / $4000+ for teacher expenses" with no framing of what a donation buys. |
| 9 | **No transparency page.** | 501(c)(3) status and EIN are buried in an About paragraph. US donors look for this explicitly. |
| 10 | **Broken/empty routes** (`/enroll`, `/cart`, `/checkout`, `/dashboard`, `/student-registration`, `/instructor-registration`). | Dead ends, crawl waste, confusion. |
| 11 | **No visible trust markers** — no ratings, no student count in hero, no partner logos, no "free trial" promise above the fold. | |
| 12 | **No time-zone communication.** | We teach from Bangladesh to North America. Class times are the #1 practical objection and we never address it. |

---

## 3. Reference research

Sites studied: Studio Arabiya (studioarabiya.com), Al-Muhammadi Academy, Online Quran Academy US, Super Muslim Academy, Miftaah Institute (miftaah.org), Qalam Seminary (qalamseminary.com), and the general category of US online madrasa / weekend Islamic school sites.

### 3.1 Two distinct patterns exist in this category

**Pattern A — the commercial academy** (Studio Arabiya, Al-Muhammadi, Super Muslim). Optimised hard for enrolment. Price visible above the fold. Free trial as the primary CTA. Teacher credentials used as the differentiator ("Al-Azhar certified", "verified Ijazah"). Heavy testimonial density. Course cards with age tags and "Starting at $X/month". A student portal is the product, and it is marketed as such.

**Pattern B — the nonprofit institute** (Miftaah, Qalam). Optimised for authority and giving. Scholar biographies are long and reverent. Donation is framed through *sadaqah jariyah* and scholarship funds. Tax ID printed in the footer of every giving page. Programs, not products. Peer-to-peer fundraising pages. Milestone/history timelines.

**Ilm-ul-Quran USA sits in both.** We enrol like Pattern A and we are funded like Pattern B. The new site must run both narratives cleanly and never let the donation ask contaminate the enrolment path (a parent researching Hifz classes should not feel they are being solicited).

### 3.2 Feature matrix

Legend: ● present · ◐ partial · ○ absent

| Feature | Studio Arabiya | Al-Muhammadi | Miftaah / Qalam | **IQU today** | **IQU target** |
|---|:--:|:--:|:--:|:--:|:--:|
| Free trial offer above the fold | ● | ● | ○ | ◐ | **●** |
| Price shown on course card | ● | ● | ○ | ○ | **●** (from-price + scholarship note) |
| Social proof in hero (student count, rating) | ● | ● | ○ | ○ | **●** |
| Age/level tags on course cards | ● | ● | ○ | ○ | **●** |
| Instructor profile with credential + bio | ● | ● | ● | ○ | **●** |
| Female-tutor pathway signposted | ● | ● | ○ | ◐ | **●** |
| Time-zone / scheduling reassurance | ● | ● | ○ | ○ | **●** |
| Student portal (schedule, reports, certificates) | ● | ◐ | ◐ | ○ | **●** |
| Parent view of a child's progress | ● | ○ | ○ | ○ | **●** |
| Report cards & certificates | ● | ○ | ○ | ○ | **●** |
| 4-step "How it works" | ● | ● | ○ | ◐ | **●** |
| Partner / trust logos | ● | ○ | ● | ○ | **●** |
| Testimonials with name & photo | ● | ● | ● | ◐ | **●** |
| Blog / articles | ● | ● | ● | ○ (draft) | **●** |
| 501(c)(3) status & Tax ID surfaced | ○ | ○ | ● | ◐ | **●** |
| Sadaqah jariyah donation framing | ○ | ○ | ● | ○ | **●** |
| Recurring / monthly sustainer giving | ○ | ○ | ● | ◐ (Zeffy) | **●** |
| Scholarship fund with named impact | ○ | ○ | ● | ◐ (Zakat coupon) | **●** |
| Institutional history timeline | ◐ | ○ | ● | ◐ | **●** |
| Org / school B2B offering | ● | ○ | ○ | ○ | Phase 2 |
| Mobile apps | ● | ○ | ○ | ○ | Out of scope |
| Recitation competition / events | ● | ○ | ● | ○ | Phase 2 |

### 3.3 What to borrow, and what to deliberately avoid

**Borrow**

- Studio Arabiya's course card anatomy: thumbnail → featured badge → title → audience tags (Kids / Teens / Adults) → from-price. It is instantly scannable and we have every one of those fields already in our course meta.
- Studio Arabiya's four-step *How it works* (Pick a course → Get matched with a teacher → Start learning online → Reports & certificates). It answers the parent's real question, which is not "what do you teach" but "what happens after I click".
- The "Track your progress with reports & certificates" section. This is the strongest retention argument in the category and maps exactly onto Sabaq/Sabqi/Manzil tracking, which we can do better than anyone because we actually teach it that way.
- Miftaah's giving structure: named funds, a hadith anchor, recurring sustainer tier, tax ID always adjacent to the ask.
- Qalam's scholarship framing — "your donation goes directly to the scholarship fund" — which is literally true for us, since half our students study free.

**Avoid**

- The SEO-farm aesthetic of the mid-tier academies: stock photos of a boy at a laptop, five stacked CTA bars, keyword-stuffed body copy. It reads as untrustworthy to exactly the parent we want.
- Studio Arabiya's density. Their homepage carries a promo banner, a competition banner, a sale banner, nine course cards, a certificate gallery, four feature blocks, five testimonials, nine partner logos, a four-step process, app badges and a newsletter — in one scroll. We will carry the same information architecture at roughly half the density.
- Countdown timers, "only 3 seats left", and other artificial scarcity. Wrong for a charity.
- Sliders and carousels for primary content. Everything important should be reachable without interaction.

---

## 4. Visual direction

Three directions were requested. All three are developed below; a recommendation follows. The designer should pick one and take it all the way — these are not meant to be blended arbitrarily.

### Direction A — "Sakīna" · Calm modern EdTech

**Idea:** the reassurance of a well-run school. Quiet, generous, uncluttered. The design gets out of the way and lets photography of real students and teachers carry the emotion.

| Token | Value |
|---|---|
| Ink | `#101C1A` |
| Paper | `#FBFAF7` |
| Surface | `#F1F3F0` |
| Primary | `#0E6B5C` (deep teal-green) |
| Accent | `#C9A227` (muted gold, used only for emphasis) |
| Muted | `#6B7A76` |

- **Display type:** a humanist sans with real character — Bricolage Grotesque or Gambetta. **Body:** Source Sans 3. **Arabic:** Kitab or Amiri Quran.
- **Radius:** 12–16 px. **Shadows:** almost none; separation by tone, not elevation.
- **Signature element:** a thin gold *mīm* rule — a hairline that terminates in a small circular terminal — used as the section divider throughout. Derived from Arabic letterform, not applied ornament.
- **Best for:** conversion. Lowest risk. Reads international and modern.
- **Risk:** if executed loosely it will look like any SaaS product.

### Direction B — "Nūr" · Warm Islamic geometric

**Idea:** the visual language of the mosque courtyard — geometry, warmth, light through a screen.

| Token | Value |
|---|---|
| Ink | `#1E1710` |
| Paper | `#F7F1E6` |
| Surface | `#EDE2CE` |
| Primary | `#0B5D4E` |
| Accent | `#B9892F` (brass) |
| Deep | `#3E2C1C` (walnut) |

- **Display:** a serif with Arabic sympathy — Vollkorn or Marcellus. **Body:** Inter Tight or Public Sans. **Arabic:** Amiri.
- **Signature element:** an eight-point *khatam* girih tile rendered as a 1 px lattice, used at extremely low opacity as a section background and at full strength as a masking shape for hero and instructor portraits.
- **Motion:** the lattice draws itself in on first load, once, then rests.
- **Best for:** emotional connection, donation pages, Ramadan campaigns.
- **Risk:** ornament is easy to overdo and can read as dated or as "generic Islamic clip-art" if the geometry is not constructed properly. Only choose this if the designer will actually build the girih grid correctly.

### Direction C — "Riwāq" · Premium editorial institution

**Idea:** the arcade of a madrasa — columns, rhythm, restraint. Looks like a seminary that has existed for a century, not a startup.

| Token | Value |
|---|---|
| Ink | `#14231F` |
| Paper | `#FCFCFA` |
| Surface | `#E8EAE4` |
| Primary | `#0A4034` (dark forest) |
| Accent | `#A8451F` (burnt sienna, sparingly) |
| Rule | `#C8CCC2` |

- **Display:** a high-contrast serif — Newsreader or Fraunces — set large, tight, and confident. **Body:** a workhorse sans, IBM Plex Sans. **Arabic:** Kitab.
- **Structure:** an explicit 12-column grid with visible hairline rules. Generous type scale. Sections separated by whitespace and rules, never by coloured bands.
- **Signature element:** the **riwāq column** — a repeating arch motif used as the container for instructor portraits and as the frame for the hero image. Arches derived from a single geometric construction and reused at three scales.
- **Best for:** authority, scholar credibility, donor confidence.
- **Risk:** can feel cold to a parent of a six-year-old. Needs warm photography to counterbalance.

### 4.1 Recommendation

**Direction C ("Riwāq") as the base, with Direction A's spacing discipline and one borrowed element from B.**

Reasoning: our differentiator is not price or convenience — those we lose to the commercial academies. Our differentiator is that the teaching is done by qualified Huffaz, Ulama and Mufti, and that we are a real 501(c)(3) that gives half its seats away. That is an *institutional* story, and institutional stories are told in editorial layouts. Direction C says "seminary". Direction A says "app". Direction B says "mosque".

Concretely:
- Base palette, type scale, grid and rules from **C**.
- Component density, radius and form design from **A** — nobody should have to fill in a form set in a display serif.
- Borrow exactly one element from **B**: the girih lattice, used only in two places — behind the donation hero, and as the empty-state graphic in the portal.

The designer should still present C at high fidelity for one hero and one interior page before we commit.

---

## 5. Design system foundations

Values below assume Direction C is selected. If another direction wins, substitute the palette and typefaces; everything structural stays.

### 5.1 Colour tokens

| Token | Hex | Use |
|---|---|---|
| `--ink` | `#14231F` | Body text, headings |
| `--ink-muted` | `#5A6862` | Secondary text, captions |
| `--paper` | `#FCFCFA` | Page background |
| `--surface` | `#E8EAE4` | Cards, wells, table stripes |
| `--surface-2` | `#F4F5F1` | Nested surfaces |
| `--primary` | `#0A4034` | Primary buttons, links, active nav |
| `--primary-hover` | `#0D5443` | Hover state |
| `--accent` | `#A8451F` | One emphasis per screen, maximum |
| `--rule` | `#C8CCC2` | Hairlines, borders, dividers |
| `--success` | `#1F6F4A` | Attendance present, payment received |
| `--warning` | `#8A6415` | Pending, awaiting review |
| `--danger` | `#8C2F1E` | Absent, failed payment, destructive |
| `--info` | `#1F5A75` | Neutral system messages |

Contrast requirement: all text/background pairs must clear **WCAG AA (4.5:1)**; large display text may sit at 3:1. Do not use `--accent` for body text.

### 5.2 Typography

| Role | Face | Sizes |
|---|---|---|
| Display | Newsreader (or Fraunces), 500/600 | 64 / 48 / 36 px |
| Heading | Newsreader 600 | 30 / 24 / 20 px |
| Body | IBM Plex Sans 400 | 18 / 16 px |
| UI / label | IBM Plex Sans 500 | 14 / 13 px |
| Data / mono | IBM Plex Mono 400 | 14 px — invoices, IDs, transaction refs |
| Arabic | Kitab / Amiri Quran | +2 px optical size vs Latin at same rank |

Rules:
- Line height 1.15 for display, 1.6 for body.
- Measure capped at **68 characters** for body prose.
- Sentence case everywhere except eyebrow labels, which are uppercase with 0.08em tracking.
- Arabic and Latin never share a line box; Quranic text sits in its own block with `dir="rtl"` and 2.0 line height.
- Never set a form label, button or table header in the display serif.

### 5.3 Spacing, grid, radius

- Base unit **4 px**. Scale: 4, 8, 12, 16, 24, 32, 48, 64, 96, 128.
- Desktop grid: 12 columns, 1200 px max content width, 24 px gutter, 1440 px max container.
- Portal grid: 12 columns, fluid, 264 px fixed sidebar.
- Section vertical rhythm: 96 px desktop / 64 px tablet / 48 px mobile.
- Radius: 4 px (inputs, chips), 8 px (buttons), 12 px (cards), 999 px (avatars, pills). Arch-framed portraits are the one exception.
- Elevation: only two levels — `0 1px 2px rgba(20,35,31,.06)` for cards, `0 8px 24px rgba(20,35,31,.12)` for overlays.

### 5.4 Breakpoints

| Name | Width | Notes |
|---|---|---|
| `sm` | ≥ 480 | Large phone |
| `md` | ≥ 768 | Tablet; nav collapses below this |
| `lg` | ≥ 1024 | Small laptop; portal sidebar appears |
| `xl` | ≥ 1280 | Desktop |
| `2xl` | ≥ 1536 | Max container caps here |

Mobile-first. **Design mobile and desktop for every screen in Section 7** — over 60% of our traffic is mobile and enrolment is overwhelmingly done on a phone.

### 5.5 Motion

- Durations: 120 ms (micro), 200 ms (default), 320 ms (page/panel).
- Easing: `cubic-bezier(0.2, 0, 0, 1)`.
- Scroll reveals: fade + 12 px rise, once, staggered 60 ms. Never on text the user is reading.
- All motion must respect `prefers-reduced-motion: reduce` — reduce to opacity only.
- No parallax, no auto-playing carousels, no scroll-hijacking.

### 5.6 Imagery

- **Photography beats illustration.** Every stock image of a generic child at a laptop must be replaced with a real student or instructor photo. Where we do not have one yet, use the arch-framed initial-monogram placeholder rather than stock.
- Instructor portraits: square source, 1:1, framed in the riwāq arch mask, warm neutral background.
- Do not depict the Quranic text as decoration. Any Quran or Hadith quotation appears as typography with a full source citation, never overlaid on a photo.
- Illustration, where used, is line-only in `--primary` at 1.5 px — no filled cartoon styles.
- All images: AVIF/WebP with fallback, explicit width/height to prevent CLS, `priority` only on the hero.

### 5.7 Iconography

Single family, 1.5 px stroke, 24 px grid — Lucide or Phosphor. No mixed sets. Icons never carry meaning alone; always paired with a label except in dense portal toolbars, where they need `aria-label` plus tooltip.

---

## 6. Global components

### 6.1 Header

Two variants.

**Public header** — sticky, 72 px tall, `--paper` background, 1 px `--rule` bottom border that only appears after 40 px of scroll.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [LOGO]   Courses ▾  Programs ▾  Instructors  About ▾  Donate    [Log in] │
│                                                        [ Start free trial ]│
└──────────────────────────────────────────────────────────────────────────┘
```

- **Courses ▾** — mega panel, 3 columns: *By level* (Noorani Qaidah, Nazirah, Hifz, Quranic Arabic) · *By age* (Kids 5–12, Teens, Adults, Sisters only) · a promoted card for the current flagship program.
- **Programs ▾** — Summer Ilm Camp, Weekend Islamic School, One-on-One, Family Plan.
- **About ▾** — Our story, Why choose us, Transparency & finances, Blog, Contact.
- **Donate** — text link in `--accent`, never a filled button. The filled button real estate belongs to enrolment.
- Primary CTA: **Start free trial** (filled `--primary`). Secondary: **Log in** (ghost).
- A slim announcement bar may sit above the header for one campaign at a time. Dismissible, remembered for 30 days, never more than one line.

**Mobile:** logo left, hamburger right. Drawer slides from the right, full height, accordion sections, both CTAs pinned to the bottom of the drawer. A persistent bottom bar on course and program pages carrying **Start free trial**.

**Portal header** — 64 px, `--surface` background, breadcrumb left, notification bell, role/child switcher, avatar menu right.

### 6.2 Footer

Four columns plus a legal strip.

| Column | Contents |
|---|---|
| 1 — Identity | Logo, one-line mission, 501(c)(3) line with EIN 93-3902300, address (100 N Central Expy Suite 914, Richardson, TX 75080), phone, email |
| 2 — Learn | All courses, Summer Ilm Camp, Weekend School, Hifz program, Quranic Arabic, Free trial |
| 3 — Organisation | About, Instructors, Transparency & finances, Impact, Blog, Careers / Teach with us, Contact |
| 4 — Support us | Donate, Monthly sustainer, Zakat scholarship fund, Sponsor a student, Zeffy giving page |

Legal strip: © year AL HASANAH FOUNDATION · Privacy · Terms · Refund policy · Accessibility statement. Social icons right. Language toggle if/when Bangla is added.

Footer newsletter sits above the columns as its own band — one input, one button, one line of consent text. This replaces the current orphaned `/enroll` page.

### 6.3 Core component inventory

The designer should deliver each of these as a component with all states (default / hover / focus-visible / active / disabled / loading / error / empty).

**Content**
`CourseCard` · `ProgramCard` · `InstructorCard` · `TestimonialCard` · `StatCard` · `ArticleCard` · `FAQAccordion` · `StepList` · `Timeline` · `QuoteBlock` (Quran/Hadith with citation) · `LogoWall` · `ImpactMeter`

**Navigation**
`Header` · `MegaMenu` · `MobileDrawer` · `Breadcrumb` · `Pagination` · `TabBar` · `SidebarNav` · `AnnouncementBar` · `StickyCTA`

**Form**
`TextField` · `Select` · `Combobox` · `RadioGroup` · `CheckboxGroup` · `DatePicker` · `TimeSlotPicker` (timezone-aware) · `PhoneField` (country code) · `FileUpload` · `CouponField` · `AmountSelector` · `Stepper` (progress) · `FormSummary`

**Feedback**
`Toast` · `InlineAlert` · `Modal` · `Drawer` · `Tooltip` · `Skeleton` · `EmptyState` · `ErrorState` · `ConfirmDialog`

**Data (portal)**
`DataTable` (sort, filter, paginate, empty, loading) · `AttendanceGrid` · `ProgressRing` · `SabaqTracker` · `ScheduleCalendar` · `InvoiceRow` · `CertificateCard` · `ChildSwitcher` · `StatusBadge`

**Attendance & verification** — all new, see Section 7D
`JoinClassButton` (armed / disabled / live / ended states) · `NextClassCard` · `LiveStatusDot` (running / waiting / no-show / not started / completed) · `OverlapBar` (teacher interval, student interval, overlap band) · `AttendanceStatusBadge` (present / partial / technical / absent / no-show / not held) · `ClassRecordForm` · `ReadOnlyFact` (system-supplied value the user cannot edit) · `DualClock` (Dhaka + student time zone) · `FlagRow` · `OverrideRequestDialog` · `DisputeLink` · `ConfirmWeekCard` · `VerifiedMinutesMeter`

### 6.4 Button system

| Variant | Use | Style |
|---|---|---|
| Primary | One per view. Enrol, Donate, Save. | Filled `--primary`, white text, 8 px radius, 44 px min height |
| Secondary | Alternative action | 1 px `--primary` outline, transparent fill |
| Ghost | Tertiary, in-table actions | Text only, underline on hover |
| Danger | Destructive | Filled `--danger`, always behind a confirm dialog |
| Link | Inline in prose | `--primary`, underlined, 2 px offset |

All buttons: 44 × 44 px minimum hit area, visible focus ring (2 px `--accent`, 2 px offset), loading state replaces the label with a spinner and keeps the width fixed.

---

## 7. Page specifications

### 7.0 Route map

**Public**

```
/                              Home
/courses                       Course catalog
/courses/[slug]                Course detail
/programs/summer-ilm-camp      Summer Ilm Camp landing
/programs/weekend-school       Weekend school landing
/programs/one-on-one           One-on-one landing
/instructors                   Instructor directory
/instructors/[slug]            Instructor profile              NEW
/about                         Our story
/about/why-us                  Why choose us                   NEW
/about/transparency            501(c)(3), finances, governance NEW
/impact                        Impact & annual numbers         NEW
/donate                        Give
/donate/sustainer              Monthly sustainer               NEW
/donate/zakat                  Zakat scholarship fund          NEW
/results/summer-2026           Camp results
/blog                          Articles
/blog/[slug]                   Article
/faq                           FAQ                             NEW
/contact                       Contact
/teach                         Instructor recruitment          NEW
/legal/privacy, /legal/terms, /legal/refund, /legal/accessibility
```

**Funnel**

```
/enroll                        Multi-step enrolment (replaces newsletter page)
/enroll/[step]                 1 student · 2 course · 3 schedule · 4 fee · 5 review
/enroll/confirmation           Thank-you + what happens next
/scholarship                   Zakat scholarship application    NEW
```

**Portal** (authenticated)

```
/login  /signup  /forgot-password  /reset-password
/dashboard                     Role router
/s/overview  /s/schedule  /s/classroom/[sessionId]  /s/progress  /s/homework
/s/reports   /s/certificates  /s/billing  /s/feedback  /s/settings

/p/overview  /p/children/[id]  /p/attendance  /p/reports  /p/billing
/p/messages  /p/confirm/[week]        (weekly confirmation landing)  NEW

/t/today     /t/classes  /t/roster/[classId]  /t/record/[sessionId]   NEW
/t/homework  /t/availability  /t/summary  /t/feedback  /t/settings

/a/live      Live board                      NEW
/a/flags     Exceptions queue                NEW
/a/registrations  /a/students  /a/instructors  /a/courses  /a/coupons
/a/donations  /a/scholarships
/a/reports/students   /a/reports/teachers    NEW
/a/feedback  /a/settings
```

**Legacy note.** The IQU SMS v1.0 specification described a teacher screen with green *Join* / red *Not Join* buttons per student and hand-typed start and end times. Those controls are removed in this version. Where a designer sees them in older material, they are superseded by Section 7D.

Every removed legacy URL gets a 301: `/enroll-for-free` → `/enroll`, `/summer-program` → `/programs/summer-ilm-camp`, `/cart` `/checkout` `/student-registration` `/instructor-registration` → 410 Gone.

---

### 7.1 Home — `/`

**Purpose.** Convince a parent in under 15 seconds that this is a real institution with qualified teachers, that trying it costs nothing, and that scheduling from North America is solved.

**Section order**

**1 · Hero**
- Eyebrow: `ONLINE QURAN EDUCATION · SERVING FAMILIES IN THE US & CANADA`
- H1: *Learn the Qur'an with qualified Huffaz, Ulama and Mufti — live, one-on-one, from home.*
- Sub: One sentence naming the three concrete things: live one-on-one or small batch, female tutors for girls and children, first month free.
- CTAs: **Start free trial** (primary) · **Browse courses** (secondary)
- Trust strip directly beneath, one line: `250+ students taught · 25+ qualified instructors · 501(c)(3) nonprofit · Nearly half of our students study free`
- Media: real photograph of a class in progress, framed in the riwāq arch. Not stock.
- **Layout:** 7/5 split desktop, image right, arch mask. Stacked on mobile with image below the CTAs so the fold stays text-first.
- **State:** hero image `priority`, explicit dimensions, LQIP blur placeholder. This is the CLS-critical element — the current site has a documented CLS problem here.

**2 · The scheduling answer** — **NEW**
The objection nobody else addresses. A compact three-column band:
- *Your time zone, not ours.* Classes run 6 AM–11 PM across ET/CT/MT/PT.
- *Choose your days.* 2, 3, 4 or 5 days a week, 30 or 45 minute sessions.
- *Never miss a class.* Free make-up sessions.
Small live widget: "It's **[current time]** in Dallas — we have instructors teaching right now." Renders client-side, degrades to static text.

**3 · Courses**
- H2: *Programs for every stage*
- Filter chips: All · Kids 5–12 · Teens · Adults · Sisters only
- Grid of 6 `CourseCard`, 3-up desktop / 2-up tablet / 1-up mobile with horizontal scroll
- CourseCard anatomy: thumbnail (16:9) → level badge → title → one-line outcome → meta row (duration · sessions/week · languages) → from-price with `Scholarships available` note → `Learn more`
- CTA below: **See all courses**

**4 · How it works**
Four steps, horizontal on desktop with a connecting rule, vertical timeline on mobile:
1. *Tell us about your student* — 3-minute form, no payment.
2. *We match you with a teacher* — by level, gender preference, language and schedule.
3. *Take a free trial class* — meet the teacher before committing.
4. *Start learning, track progress* — weekly Sabaq/Sabqi/Manzil reports.

**5 · Instructors**
- H2: *Taught by scholars, not by volunteers*
- 6 `InstructorCard` in an arch-framed row, each with photo, name, credential line (e.g. *Mufti · Dars-e-Nizami · 8 years teaching*), specialisation tags, languages
- CTA: **Meet all 20 instructors**
- **This section must not ship with grey placeholders.** If photos are unavailable, use monogram placeholders in `--surface` with the arch mask.

**6 · Progress & reports** — **NEW**
- H2: *You'll always know how your child is doing*
- Left: prose on weekly reports, attendance, Sabaq/Sabqi/Manzil tracking, term certificates.
- Right: a real screenshot of the parent dashboard in a device frame.
- CTA: **See a sample report**

**7 · Testimonials**
Three cards with photo, full name, city/state, program studied, and a specific outcome ("finished Noorani Qaidah in 11 weeks"). Vague praise is worse than no testimonial. Below: rating summary if we have one, otherwise omit rather than fabricate.

**8 · Nonprofit band**
Distinct background (`--surface`), girih lattice at 4% opacity. Short: who we are, EIN, the fact that nearly half of students pay nothing, and two links — **How we're funded** (→ `/about/transparency`) and **Support a student** (→ `/donate`). Text link, not a competing filled button.

**9 · FAQ preview**
Six accordion items answering: cost, free trial, time zones, female teachers, minimum age, what device is needed. CTA: **All questions**.

**10 · Final CTA**
Full-width `--primary` band. Single headline, single button, one line of reassurance ("No card required. Cancel any time.").

**States.** Course grid: skeleton cards while loading, empty state if a filter returns nothing ("No courses match that filter yet — see all courses"). Testimonials and instructors must render server-side.

**Mobile.** Sections 2 and 6 collapse to stacked cards. Sticky bottom CTA appears after the hero leaves the viewport.

**SEO.** `Organization` + `EducationalOrganization` + `FAQPage` schema. H1 once. Hero image is the LCP element — budget under 2.0 s.

---

### 7.2 Course catalog — `/courses`

**Purpose.** Let a parent narrow to the right program in two clicks.

**Sections**
1. **Page header** — H1 *Courses*, one-sentence intro, and the count ("6 programs · new courses each term").
2. **Filter bar** — sticky below header. Facets: Level (Beginner / Intermediate / Advanced), Age (Kids 5–12 / Teens 13–17 / Adults), Format (One-on-one / Small batch / Weekend / Seasonal), Language (English / Bengali / Arabic / Urdu), Teacher gender (Any / Female only). Active filters shown as removable chips with a **Clear all**.
3. **Results grid** — 3-up desktop, `CourseCard` as specified in 7.1. Sort control: Recommended / Newest / Duration.
4. **Comparison strip** — **NEW.** Checkbox on each card allows up to 3 courses to be compared in a bottom drawer showing duration, sessions, languages, certificate, price.
5. **Not sure which course?** — a short 4-question quiz linking to `/enroll` with the answers pre-filled. Powerful and cheap to build.
6. **CTA band** — free trial.

**States.** Loading skeletons · empty result with the active filters listed and a Clear all · error state with retry. Filter state must live in the URL query string so results are shareable and indexable.

**Mobile.** Filters move into a bottom sheet triggered by a **Filter (3)** button. Grid becomes 1-up.

---

### 7.3 Course detail — `/courses/[slug]`

**Purpose.** Answer every question a parent has, then convert.

**Sections**

**1 · Course hero**
- Breadcrumb · level badge · H1 course title · one-paragraph outcome statement (what the student will be able to do at the end, not what the course "covers")
- Meta row: duration · lessons · sessions per week · languages · certificate yes/no · age range
- Price block: `From $X / month` + `Zakat scholarships available — apply during enrolment`
- CTAs: **Start free trial** (primary) · **Ask a question** (secondary, opens WhatsApp/contact drawer)
- Right: course image, or a 60-second teacher intro video if available

**2 · Sticky sub-navigation** — Overview · Curriculum · Teachers · Schedule · Fees · Reviews · FAQ. Scroll-spy highlights the active section. On mobile this becomes a horizontally scrollable tab bar. *(The current site uses a click-tab widget that hides content from crawlers — replace it with anchored sections that are all present in the DOM.)*

**3 · Overview** — Who it's for · What you'll learn (5–7 bullets, outcome-phrased) · What you need (device, internet, book list) · Prerequisites.

**4 · Curriculum & method** — Module accordion. Each module: title, duration, lesson count, description, and — for Quran courses — the Sabaq/Sabqi/Manzil breakdown. This is our differentiator; make it explicit and visible.

**5 · Your teachers** — Assigned instructors as `InstructorCard`s with a note that final matching happens after the trial.

**6 · Schedule & availability** — **NEW.** A time-zone-aware grid showing which slots have openings this term. Selector for ET/CT/MT/PT. Even a coarse "morning / afternoon / evening" availability view removes the biggest objection.

**7 · Fees & scholarships** — **NEW.** Three columns: standard monthly fee, sibling discount, Zakat scholarship. Plain language on the subsidy: what it covers, who qualifies, how to apply. Link to `/scholarship`.

**8 · Reviews** — Real reviews only, with name, location and date. If a course has none, show the section as an empty state inviting current students to review — never a fabricated "20 Reviews" badge, which is what the current site displays.

**9 · Related courses** — 3 cards, "Students often continue with…".

**10 · Course FAQ** — 6–8 items specific to this course.

**11 · Sticky enrolment bar** — appears on scroll past the hero. Course name, from-price, **Start free trial**. Desktop: right rail card. Mobile: bottom bar.

**Schema.** `Course` + `Offer` + `AggregateRating` (only if genuine) + `FAQPage`. Rank Math templates already exist for this and should be ported.

---

### 7.4 Instructor directory — `/instructors`

**Purpose.** Convert our biggest asset into visible credibility.

**Sections**
1. **Header** — H1 *Our instructors*, sub: *Qualified Huffaz, Ulama and Mufti — every teacher is vetted for both scholarship and teaching ability.* Count: *20 instructors · 6 languages*.
2. **Filters** — Specialisation (Hifz / Tajweed / Nazirah / Arabic / Islamic Studies) · Language · Gender · Availability.
3. **Grid** — `InstructorCard`, 4-up desktop / 2-up tablet / 1-up mobile.
   Card anatomy: arch-framed portrait → name → title (Mufti / Mawlana / Hafez / Ustadha) → credential line → specialisation chips → language chips → *View profile*.
4. **Leadership callout** — Director (Mufti Muhammad Faijullah) and Education Secretary shown larger, above the general grid.
5. **Teach with us** — recruitment CTA → `/teach`.

**Critical note for the designer.** The card must be designed to look complete even when the photo is missing, because roughly half our instructor records have no portrait today. Design the monogram fallback as a deliberate element, not an accident.

---

### 7.5 Instructor profile — `/instructors/[slug]` — **NEW**

**Sections**
1. **Profile header** — arch-framed portrait, name (Latin + Arabic), title, one-line positioning, chips for specialisation / languages / years teaching, CTA **Request this teacher** (pre-fills enrolment).
2. **Biography** — 3–5 paragraphs. Where trained, under whom, what they specialise in, teaching philosophy.
3. **Credentials** — structured list: institution, qualification, year. Ijazah/sanad chain where applicable, rendered as a small vertical lineage list.
4. **Courses taught** — `CourseCard` row.
5. **Recitation sample** — **NEW.** Audio player, 30–60 s. Single strongest conversion element on this page for a Quran teacher. Design a minimal player: play/pause, waveform or progress line, duration.
6. **Student feedback** — 2–3 quotes.
7. **Availability** — general slots, time-zone aware.

**Schema.** `Person` + `worksFor` Organization. Rank Math Person schema already configured — port it.

---

### 7.6 About — `/about`

**Sections**
1. **Header** — H1 *Our story*, one-line mission.
2. **Origin** — how and why IQU began, written as narrative not as a bulleted mission statement.
3. **Mission & vision** — two cards, restrained. Reuse the existing "History" and "Goal" copy but tighten it by half.
4. **Timeline** — **NEW.** Founding → first cohort → 501(c)(3) recognition → first Summer Ilm Camp → 250 students. Vertical on mobile, horizontal rule-based on desktop. Miftaah's timeline is the reference.
5. **By the numbers** — 4 `StatCard`: students taught, instructors, countries reached, free/subsidised seats. Numbers count up once on scroll (respecting reduced motion).
6. **Leadership & staff** — photo grid with role and one-line bio.
7. **Legal identity** — pulled out of body prose into a bordered card: Legal name, operating name, tax status, EIN, IRS recognition, address. Link to `/about/transparency`.
8. **CTA** — split: *Enrol a student* | *Support our work*.

### 7.7 Why choose us — `/about/why-us` — **NEW**

Six differentiators, each a full band alternating image side:
1. Teachers with real sanad, not gig tutors
2. Sabaq / Sabqi / Manzil — the method Hifz is actually taught with
3. Female instructors for girls and young children
4. Your time zone, your days
5. Weekly written progress reports to parents
6. A nonprofit — nobody is turned away for cost

Close with a comparison table: *Ilm-ul-Quran USA* vs *typical online tutoring* vs *local weekend school* on the axes parents care about — teacher qualification, one-on-one, flexibility, cost, progress reporting, safeguarding.

### 7.8 Transparency & finances — `/about/transparency` — **NEW**

US donors look for this before giving. Sections:
1. **Legal status card** — 501(c)(3), EIN 93-3902300, state of incorporation, IRS determination letter (PDF download), address.
2. **Where the money goes** — a single honest chart: teacher stipends / platform & technology / scholarships / administration. Use real Zeffy-derived figures.
3. **Fee policy** — plain-language explanation that we are not free any more, that nearly half of students still pay nothing, and how the subsidy is decided.
4. **Zakat policy** — who administers it, what it may be spent on, how eligibility is verified.
5. **Governance** — board or trustees, conflict-of-interest statement.
6. **Annual reports** — downloadable PDFs, one card per year.
7. **Contact for donors** — a named person and an email.

### 7.9 Impact — `/impact` — **NEW**

Driven live from `iqu_registrations` and `iqu_zeffy_payments`.
1. Hero counter band: students enrolled, classes delivered, scholarship seats funded, donors this year.
2. **What your donation buys** — three cards: $25 = one week of classes for one student · $100 = one month · $1,200 = a full year of a scholarship seat. *(Amounts to be confirmed against real unit cost before publication.)*
3. Student stories — 3 long-form cards with photo, name, before/after.
4. Donor map or list of supporting cities — optional, only if the data is clean.
5. CTA → `/donate`.

### 7.10 Donate — `/donate`

**Purpose.** Convert a supporter in one screen, and make repeat/recurring giving the default.

**Sections**

**1 · Hero** — girih lattice background (the one borrowed element from Direction B).
- Hadith anchor, properly cited: *"When a person dies, their deeds end except three: a continuing charity, beneficial knowledge, or a righteous child who prays for them."* — Sahih Muslim. Arabic above, English below, source line in `--ink-muted`.
- H1: *Fund a seat. Teach a child. Earn a sadaqah jariyah.*
- Right: the donation widget itself, above the fold.

**2 · Donation widget** — the most important component on the site.
- Frequency toggle: **Monthly** (default, pre-selected) / One-time
- Amount chips: $25 · $50 · $100 · $250 · Other. Each chip carries its impact label underneath.
- Fund selector: Where needed most (default) · Teacher stipends · Zakat scholarship fund · Summer Ilm Camp · Technology
- Zakat checkbox: *This is my Zakat* — reveals a one-line note on how Zakat is segregated
- Dedication field (optional): *In memory of / On behalf of*
- Tax-deductible line + EIN, immediately below the button — never in the footer only
- Button: **Give $100 monthly** — the label must restate the choice
- Zeffy handoff: if the transaction completes off-site, the design must show a clear interstitial explaining that they are moving to our secure giving partner, and the return page must be branded.

**3 · Impact strip** — three numbers with the honest framing already used on the current site: nearly half of students receive classes at no cost, the rest are significantly subsidised.

**4 · Sustainer tier** — **NEW.** *Become a monthly sustainer.* Named tiers with what each funds. Benefits: quarterly impact report, name on the supporters page (opt-in), du'a from the students.

**5 · Other ways to give** — Zakat, sponsor a specific student, employer matching, stock/DAF, cheque by post. Each a small card.

**6 · Transparency reassurance** — 3 links: 501(c)(3) status · Where the money goes · Annual report.

**7 · Donor FAQ** — is it tax deductible, will I get a receipt, can I cancel monthly giving, is my Zakat handled correctly, can I give in CAD.

**States.** Widget: idle · validating · processing (button locked, spinner, no double-submit) · success (receipt number, email confirmation notice, share prompt) · failure (specific error, card unchanged, retry). Never lose the entered amount on error.

**Mobile.** Widget moves directly below the hero headline, above everything else.

### 7.11 Summer Ilm Camp — `/programs/summer-ilm-camp`

A seasonal landing page. It should be designed once and re-skinned each year.

1. Hero — dates, ages, schedule, price, seats. `Summer Ilm Camp Online 2026 · Ages 5–15 · 8 weeks · 4 days a week · June 1 – July 31 · Mon–Thu, 10 AM–12 PM CDT`. Countdown to registration close is acceptable here because the deadline is real.
2. What your child will learn — 4 pillars with icons.
3. Daily schedule — a visual timetable of a typical day.
4. Curriculum by age band — three columns: 5–7, 8–11, 12–15.
5. Meet the camp teachers.
6. Last year's camp — photos, and a link to `/results/summer-2026`.
7. Fees, sibling discount, scholarship.
8. Registration form — the existing `iqu_summer_form` rebuilt as a staged flow.
9. FAQ.
10. Sticky register bar with seats-remaining if the number is real.

### 7.12 Camp results — `/results/summer-2026`

Already exists in a working form. Rebuild as:
1. Header with cohort name and exam date.
2. Search by student name or roll.
3. Leaderboard table — rank, name, age band, score, grade. Top 3 highlighted with a distinct treatment, not a trophy emoji.
4. Individual result card on row click — modal on desktop, full page on mobile — showing subject breakdown and teacher remark.
5. Download certificate (PDF) if eligible.
6. CTA — enrol for the next term.

**States.** Results not yet published (countdown + notify-me), student not found (clear message, contact link), large list (virtualised, paginated).

### 7.13 Blog — `/blog` and `/blog/[slug]`

Currently a 67 KB draft that has never shipped. Launch it.

**Index.** Featured post (large card) → category chips (Tajweed, Hifz, Parenting, Arabic, Announcements) → 3-up article grid → pagination → newsletter band.

**Article.** Max 68-character measure · H1 · author with instructor link · date and reading time · hero image · sticky table of contents on desktop · pull-quote and callout styles · Quran/Hadith `QuoteBlock` with citation · author bio card · related articles · comment section omitted for v1 · `Article` schema.

### 7.14 FAQ — `/faq` — **NEW**

Search field at the top. Categories: Getting started · Courses & curriculum · Teachers · Schedule & time zones · Fees & scholarships · Technology · Safeguarding · Donations. Accordion within each. Each answer ends with a related link. Deep-linkable anchors. `FAQPage` schema.

### 7.15 Contact — `/contact`

Keep the existing four-card structure — it works — but upgrade:
1. Four channel cards: Email · Visit · Call/Text · WhatsApp. Each with hours **stated in the visitor's own time zone**.
2. Contact form: name, email, phone, *I am a* (parent / student / donor / instructor / press), subject, message. Response-time promise stated explicitly.
3. Map — static image, loaded lazily, links out. Do not embed a live map iframe; it costs Core Web Vitals.
4. Departmental routing: admissions, donations, technical support, press.
5. Link to FAQ above the form to deflect volume.

### 7.16 Teach with us — `/teach` — **NEW**

Replaces the empty `/instructor-registration` page.
1. Hero — *Teach the Qur'an to families across North America.*
2. Why teach with IQU — stipend, training, schedule, community.
3. What we look for — qualification, Ijazah/sanad, English proficiency, internet and device requirements.
4. Hiring process — 4 steps: apply → screening → demo class → onboarding.
5. Application form — staged: personal details, qualifications, sanad, languages, availability, demo recitation upload, references.
6. Current openings.

---

## 7B. Enrolment funnel

The single highest-value design work in this project. Today it is one shortcode-rendered form writing 50 columns to `iqu_registrations`. Every field below already exists in that table — this is a redesign of presentation, not a change of data model.

### 7.17 Enrolment — `/enroll`

**Principle.** Five short steps, each answering one question, with the emotional reassurance carried in a persistent right rail. No payment is taken in this flow. The outcome is a trial class booking.

**Persistent frame**
- Slim header: logo, step progress (`Step 2 of 5`), **Save and finish later** (emails a resume link)
- Progress bar with named steps: Student · Course · Schedule · Fees · Review
- Right rail (desktop) / collapsible summary card (mobile): running summary of choices, plus one reassurance line that changes per step
- Autosave to localStorage on every field blur; restore on return with a visible "We saved your progress" notice

**Step 1 — About the student**
Fields: student first/last name · age · is the student under 18 (reveals guardian block) · guardian name, contact, WhatsApp · country of origin · country of residence · preferred language(s) · email · phone.
Reassurance: *We never share your details. No payment is required to start.*

**Step 2 — What they want to learn**
- Current Quran level: cannot read Arabic letters / reading Qaidah / reading Quran / memorising — as illustrated radio cards, not a dropdown
- Amount already memorised (conditional, only if "memorising")
- Course selection: cards, single-select, pre-filled if they arrived from a course page
- Enrolment level and course type

**Step 3 — Schedule**
The step that wins or loses the enrolment.
- Time zone: auto-detected, editable, shown prominently
- Days per week: 2 / 3 / 4 / 5
- Preferred days: multi-select day chips
- Session length: 30 / 45 / 60 min
- Time slot: a **grid of real available slots in the user's local time**, not a free-text field. Unavailable slots visibly disabled, not hidden.
- Teacher gender preference: Any / Female only — with a plain note that female instructors are available for girls and young children
- Device: computer / tablet / phone

**Step 4 — Fees**
Honest, and the emotional heart of the form.
- Show the standard fee for the chosen configuration, calculated live
- Three options as radio cards:
  1. *I can pay the standard fee*
  2. *I need a reduced fee* — reveals a short, dignified free-text reason field
  3. *I am applying for a Zakat scholarship* — reveals the Zakat declaration checkbox and eligibility note
- Coupon field with inline validation against `iqu_coupons` (valid / expired / not applicable to this course / usage limit reached — each with its own message)
- Sibling discount auto-applied if a matching guardian email already exists
- Reassurance: *Nearly half of our students study at no cost. Asking for help will never affect your child's place.*

**Step 5 — Review & submit**
- Full summary, every group with an **Edit** link that returns to that step and back
- WhatsApp updates opt-in · future-updates opt-in · WhatsApp group opt-in
- How did you hear about us (with an "Other" text field)
- Terms and privacy consent — unticked by default
- Submit: **Book my free trial class**

**Confirmation — `/enroll/confirmation`**
- Reference number, displayed large and copyable
- **What happens next**, with real timings: we review within 24 hours → we match a teacher → you receive a WhatsApp message with your trial time → your trial class
- Add-to-calendar placeholder for the trial once scheduled
- Links: join the WhatsApp group · read the parent guide · explore other courses
- Conversion tracking fires here (Meta CAPI Lead event already implemented server-side — keep the deduplication logic)

**States across the flow.** Field-level validation on blur, never on keystroke. Errors are specific and adjacent to the field. Step-level error summary at the top with anchor links, focus moved to it. Submit button locks with a spinner. Network failure preserves everything and offers retry. Duplicate email detected → offer to resume the existing application rather than creating a second row.

**Mobile.** One question group per screen. Progress bar pinned to the top. Next/Back pinned to the bottom. Numeric and email keyboards set correctly via `inputmode`. Never open a native date picker for a slot chooser.

### 7.18 Zakat scholarship application — `/scholarship` — **NEW**

Can be reached standalone or from Step 4. Sections: what the scholarship covers · who qualifies · how we verify · how Zakat funds are segregated · the application form (household situation, number of children enrolling, requested contribution, declaration) · what happens next and the decision timeline.

The tone here matters more than the layout. Nothing in this page should make a family feel audited.

---

---

## 7C. Attendance & verification engine

The largest functional change in this version, and the one that reshapes every portal screen. Read this before designing anything in Section 7D.

### 7C.1 The principle

**Attendance is measured, not reported.**

Every class runs in a Zoom meeting owned by the organization. Zoom notifies our server the moment anyone joins or leaves. From those events the system computes one number:

> **Overlap** — the number of minutes the teacher and the student were present in the meeting *at the same time*.

Not "was the student there". Not "was the teacher there". Both, together. That single number closes every gap: a teacher sitting alone scores zero, a student sitting alone scores zero, and a class that never started produces no record at all.

Three consequences shape the UI:

1. **Nobody marks attendance.** There is no attendance control anywhere in the teacher interface. Designers should treat any attendance value as a *fact the system states*, never as an input.
2. **A teacher cannot claim a class that did not happen.** If Zoom never started, the class shows as Not held.
3. **Teacher absence is surfaced as prominently as student absence.** The previous specification tracked only students who failed to join.

### 7C.2 Status model

Six states. Every attendance display in the product uses this set and no other.

| Status | Condition | Colour token | Where it surfaces |
|---|---|---|---|
| **Present** | Overlap ≥ 70% of scheduled duration | `--success` | Everywhere |
| **Partial** | Overlap 30–70% | `--warning` | Teacher states a reason |
| **Technical issue** | Overlap > 0 but < 30% | `--info` | Make-up offered; not an absence |
| **Student absent** | Teacher present, overlap 0 | `--danger` | Parent notified |
| **Teacher no-show** | Student present, overlap 0 | `--danger` | Admin alerted in 10 min |
| **Class not held** | No Zoom events at all | `--ink-muted` | Admin flag |

Thresholds are placeholders until calibrated against three weeks of live data. Design the badge component so the percentages can change without redesign.

### 7C.3 Division of responsibility

The system and the teacher never contradict each other because they answer different questions. This split is the key to the screen design.

| The system states (read-only) | The teacher supplies (editable) |
|---|---|
| Whether the class happened | The reason for an absence: Sick · Student leave · Teacher leave |
| Who joined, and when | Remarks on the student |
| How long each side stayed | What was taught (Sabaq / Sabqi / Manzil ranges) |
| Overlap in minutes | Quality rating |
| Present / Absent / Partial | Homework assigned |

**Design rule.** Read-only facts and editable fields must be visually distinct at a glance. Read-only values sit in a `ReadOnlyFact` treatment — `--surface-2` background, no border, no focus ring, a small system glyph. They must never look like a disabled input, because a disabled input implies "you could edit this if something changed".

### 7C.4 The overlap visualisation

`OverlapBar` is the signature component of the portal and appears on the class record, the parent's class detail, and the admin session drawer.

```
        6:55                                        7:32
Teacher  ├──────────────────────────────────────────┤
Student        ├────────────────────────────────────┤
              7:02
Overlap        ███████████████████████████████████     30 min
        │                                              │
     scheduled 7:00                            scheduled 7:30
```

Two horizontal tracks against a shared time axis, with the overlap band beneath in `--primary`. The scheduled window is marked with hairlines so a late start is immediately visible. On mobile the tracks stack and the axis compresses to start/end labels only.

### 7C.5 Corrections

A teacher may request a correction — for example a class that ran on WhatsApp because Zoom failed. The request states a reason and enters an admin queue. It is never applied silently.

- Entry point: a small text link on the class record, never a button competing with Save.
- The dialog states plainly what the system recorded and asks what actually happened.
- While pending, the class shows the original status with a "correction requested" marker.
- **Override rate per teacher is itself a monitored figure** and appears on the teacher report.

### 7C.6 Independent verification

Two lightweight mechanisms outside the teacher's control. Neither requires recordings, which are deliberately not part of this system.

**Weekly parent confirmation.** Each Friday the parent receives a short email listing the week's classes and total minutes, with two actions: *Yes, that's right* and *Something's wrong*. One tap. Landing page at `/p/confirm/[week]`.

**Report a problem.** Beside every completed class in the student and parent portals, a quiet text link: *Was there a problem with this class?* One click plus an optional comment. Deliberately lighter than the Feedback tab — Feedback is for considered thoughts, this is for a parent noticing that Tuesday's class ran five minutes.

Dispute rate per teacher becomes the platform's most meaningful quality signal.

### 7C.7 What the designer must not build

Carried over from the earlier IQU SMS draft and now removed. If these appear in older material, they are superseded.

| Removed | Why |
|---|---|
| Green **Join** / red **Not Join** buttons per student | Zoom determines attendance |
| **Start time** and **End time** input fields | Zoom provides exact timestamps |
| **Present / Absent** selection by the teacher | Determined by overlap |
| **Mark class as completed** | The meeting ending marks it |
| Class recording and audio audit | Not required — the two sides have opposing incentives, and disputes plus progress records cover the gap at lower cost and without a privacy burden |

---

## 7D. Portal

Authenticated. Four roles, one shell, role-specific navigation.

### 7D.1 Portal shell

- **Sidebar** 264 px, `--surface`, collapsible to 72 px icons. Bottom: help, sign out.
- **Topbar** 64 px: breadcrumb · global search (teacher/admin) · notification bell with unread dot · child switcher (parent) · avatar menu.
- **Content** max 1200 px, 32 px padding, 24 px card gap.
- **Mobile:** sidebar becomes a bottom tab bar of 4–5 destinations; the rest under "More".
- **Empty states** are a designed component — girih line illustration, one sentence, one action.
- **Live region.** Any screen showing class status subscribes to updates and re-renders without a refresh. Design a subtle transition; never a full-page flash.

### 7D.2 Authentication — `/login` · `/signup` · `/forgot-password` · `/reset-password`

Split layout: form left on `--paper`, arch-framed photograph right with a short student quote. Mobile: form only.

- **Login** — email, password with show/hide, remember me, forgot link. Errors as an inline alert above the form, never a toast.
- **Signup** — for families enrolled before the portal existed. Email, password with live strength meter and stated rules, phone, consent. Then an email-verification screen with resend and cooldown.
- **Reset** — request → sent → set new → success. Design the token-expired and token-already-used states explicitly.

### 7D.3 Student — `/s/*`

**Overview `/s/overview`**
- Greeting with the student's name, Gregorian and Hijri date.
- **`NextClassCard`** — the most prominent element on the screen. Course, teacher with photo, time in the student's zone, countdown, and **`JoinClassButton`**.
- Four stat tiles: attendance this month · current Sabaq · Manzil due · classes completed.
- This week's schedule strip with per-class status badges.
- Latest teacher remark. Homework due.

**`JoinClassButton` states** — specify all five:

| State | When | Appearance |
|---|---|---|
| Disabled | More than 10 min before start | Ghost, next class time beside it |
| Armed | 10 min before → start | Filled `--primary`, "Join class" |
| Live | Class in progress | Filled, pulsing dot, "Join now — in progress" |
| Ended | 30 min after end | Replaced by the class summary |
| No class | Nothing scheduled today | Card replaced by an empty state |

**Schedule `/s/schedule`** — week and month views, all times in the student's own zone with the zone named in the header. Per class: course, teacher, time, join control, status. Request-reschedule action. Make-up classes visually distinguished.

**Classroom `/s/classroom/[sessionId]`** — pre-class device check (camera, mic, connection) → Join → after the class the same route shows the summary: `OverlapBar`, duration, what was taught, teacher remark, homework, and the *Report a problem* link.

**Progress `/s/progress`** — the differentiator.
- `SabaqTracker`: three tracks — **Sabaq** (new lesson), **Sabqi** (recent revision), **Manzil** (long revision) — each an ayah/page range bar with completed, in-progress and pending segments.
- Hifz progress: 30-cell juz grid, colour-coded, hover reveals surah range and completion date.
- Tajweed skills rubric with per-skill teacher ratings.
- Milestone timeline.

**Homework `/s/homework`** — today's homework on the dashboard for focus; a dedicated tab for the past month. Submission is a text area plus file upload. Teacher feedback thread per item.

**Reports `/s/reports`** — monthly report cards. Card grid → detail with attendance, progress per track, teacher comments, next-month targets. PDF download.

**Certificates `/s/certificates`** — earned certificates as cards with preview, download, share. Locked certificates greyed with the criteria required.

**Billing `/s/billing`** — plan, next payment, method, invoice history, receipts. **A family on a full scholarship sees a scholarship status card, never an empty invoice table with a $0 balance.**

**Feedback `/s/feedback`** — text area plus optional file upload, submitted to admin. Carried over unchanged from the earlier specification.

**Settings `/s/settings`** — profile, photo, password, time zone, notification preferences per event type, language.

### 7D.4 Parent — `/p/*`

Same primitives, framed around oversight rather than participation. **Child switcher** in the topbar with an *All children* aggregate.

**Overview** — one row per child: next class with join control, attendance this month, current progress, latest remark, and alerts (missed classes, payment due, teacher no-show).

**Child detail `/p/children/[id]`** — the student view read-only, plus message-teacher.

**Attendance `/p/attendance`** — `AttendanceGrid`: children as rows, days as columns, cells colour-coded across the six statuses of Section 7C.2. Month navigation, export. **Sick and approved leave are excluded from the percentage denominator** — illness must not damage a child's record, and the UI should say so in a footnote.

**Reports `/p/reports`** — all children's report cards, filterable by child and month.

**Billing `/p/billing`** — consolidated across children, sibling discount as a line item, one payment method.

**Messages `/p/messages`** — threaded conversations with each child's teacher and with admin. Not live chat; a considered inbox with response-time expectations stated.

**Weekly confirmation `/p/confirm/[week]`** — the landing page from the Friday email. A summary card of the week's classes and total minutes, and two large actions. After answering, a short thank-you and a link into the portal. Must work when signed out, from a signed link.

### 7D.5 Teacher — `/t/*`

**Today `/t/today`** — the landing view. One card per class scheduled today.
- Student name and photo, course, scheduled time, session length.
- `JoinClassButton`, armed 10 minutes before.
- Live status once the meeting starts: waiting for student · in progress with elapsed time · completed.
- After the class the card becomes the class record entry point.
- **`DualClock`** — every time in the teacher portal appears in Dhaka time with the student's local time beside it. A warning banner appears in the two weeks before each North American daylight-saving change.

**Classes `/t/classes`** — assigned classes, roster size, schedule, next session.

**Roster `/t/roster/[classId]`** — student list with level, attendance rate, current Sabaq, last remark. Row opens a student drawer.

**Class record `/t/record/[sessionId]`** — replaces the removed attendance form. The most-used screen in the teacher portal.

| Field | Type | Notes |
|---|---|---|
| Attendance status | **Read-only** | System-supplied. `ReadOnlyFact` treatment. |
| Duration and `OverlapBar` | **Read-only** | Shows both sides' intervals |
| Absence reason | Dropdown | Appears **only** when the status is an absence: Sick · Student leave · Teacher leave |
| What was taught | Range fields | Sabaq, Sabqi, Manzil for Quran courses; free text otherwise |
| Quality | Rating | Teacher's assessment |
| Remarks | Text | Visible to the parent — say so under the field |
| Homework | Text + file | Feeds the student's homework tab |
| Request a correction | Text link | Opens `OverrideRequestDialog` |

**Six-hour submission window.** The record must be submitted within six hours. The form then locks and an admin must reopen it. Design three states: open with a countdown, closing soon (under one hour, `--warning`), and locked with a reopen-request link. This prevents a week of records being written from memory on a Friday evening.

**Homework `/t/homework`** — student dropdown, homework text box, file upload for a scanned worksheet or diagram, submit. Retained from the earlier specification for assigning work outside class time; inside class time it lives in the class record so the teacher does not visit a second screen after every class.

**Availability `/t/availability`** — weekly grid in Dhaka time with the corresponding US times alongside. Feeds assignment and capacity planning.

**Teaching summary `/t/summary`** — monthly statement: verified teaching minutes, classes delivered, no-shows, correction requests, records submitted on time. **Payment is calculated from verified minutes, not self-reported hours** — state this on the screen, because it is the mechanism that aligns everyone's incentives and teachers should understand it.

**Feedback `/t/feedback`** — text area plus file upload to admin. Carried over unchanged.

### 7D.6 Admin — `/a/*`

**Live board `/a/live`** — the default admin landing screen. Driven by Zoom events, genuinely live rather than dependent on anyone pressing a button.

| Indicator | Meaning |
|---|---|
| Green — Running | Both parties in the meeting now, elapsed time shown |
| Red — Student not joined | Teacher present, student absent |
| Red — Teacher no-show | Student present, teacher absent |
| Amber — Not started | Scheduled start passed, no meeting began |
| Grey — No class today | Teacher has nothing scheduled |
| Blue — Completed | Finished, with final duration |

Grouped by teacher, with assigned student names beneath each. A day summary strip across the top: scheduled · completed · running · failed.

**Flags `/a/flags`** — a queue holding only what needs a human decision. Everything else resolves itself. Teacher no-shows · classes not held · correction requests · students with three or more consecutive absences · late class records · parent-reported problems · two classes on one licence at the same time. Each row: what happened, who, when, and one primary action.

**Registrations `/a/registrations`** — `DataTable` over `iqu_registrations`. Columns: reference, name, age, course, level, schedule preference, fee preference, status, created. Filters: status, form type, course, date range, fee preference. Row drawer with the full record, admin note, and status transitions (new → reviewing → teacher assigned → trial scheduled → enrolled → declined). Bulk actions, CSV export.

**Students / Instructors / Courses `/a/students` `/a/instructors` `/a/courses`** — CRUD tables. On enrolment the system creates the student's recurring Zoom meeting and stores the meeting ID and join link; where the API is not yet connected the admin pastes a link created by hand, and everything downstream behaves identically. **Design the manual-paste path** — it is how the first months will run.

**Coupons `/a/coupons`** — over `iqu_coupons`. Code, occasion, course type, discount type and value, expiry, max uses, with a usage progress bar.

**Donations `/a/donations`** — over `iqu_zeffy_payments`. Six metric cards and the existing charts ported: total raised, donors, recurring vs one-time, average gift, refunds and disputes, campaign breakdown. Table with buyer, amount, fund, method, receipt link. Sync status indicator and manual re-sync. **This reads the existing Zeffy sync — it must not create a second, competing record of giving.**

**Scholarships `/a/scholarships`** — applications queue with approve and decline, decline requiring a reason.

**Student report `/a/reports/students`** — carried over unchanged in structure from the earlier specification. Filters by month, week, or specific dates. Days attended and missed, total class duration in minutes per day, total time and attendance percentage for the range. Exportable.

**Teacher report `/a/reports/teachers`** — the counterpart the earlier specification lacked. Same filters, per teacher: verified minutes, classes delivered, no-show count, average overlap as a percentage of scheduled duration, records submitted on time, correction requests raised, parent disputes received.

**Feedback `/a/feedback`** — dropdown selecting Student feedback or Teacher feedback. Each entry shows submitter, date and time, text, and attached file with download. Filterable by date and name, with an internal response and note field. Carried over unchanged.

**Settings `/a/settings`** — org details, fee schedule, term dates, **attendance thresholds** (editable, because they are calibrated after launch), notification templates, Zoom account and licence pool, Telegram and email integration status.

### 7D.7 Notifications

Design the in-product notification list and the email templates from the same set.

| Event | Recipient | Channel | Timing |
|---|---|---|---|
| Class starting soon | Student, teacher | Email / WhatsApp | 15 min before |
| **Teacher no-show** | Admin | Telegram | 10 min after start |
| **Student absent** | Parent, admin | Email / Telegram | At class end |
| **Class not held** | Admin | Telegram | 30 min after start |
| Class record overdue | Teacher | Email | 6 hours after class |
| Homework assigned | Student | Email | On submission |
| Weekly summary and confirmation | Parent | Email | Friday |
| Three consecutive absences | Admin | Telegram | On the third |
| Correction request | Admin | Telegram | On submission |

### 7D.8 Data the screens read from

For the designer's understanding of what is real and what must be invented.

| Store | Holds | Status |
|---|---|---|
| `iqu_registrations` | 50 fields per enrolment — student, guardian, level, schedule, fee, coupon, Zakat | **Exists, 81 rows** |
| `iqu_zeffy_payments` | Full donation sync | **Exists, 63 rows** |
| `iqu_coupons` | Discount codes | **Exists, 3 rows** |
| `classes` | Student, teacher, schedule, `zoom_meeting_id`, `zoom_join_url`, host account | New |
| `class_sessions` | One row per occurrence: scheduled and actual times, teacher minutes, student minutes, **overlap_minutes**, status, absence reason, taught content, record timestamps, override trail | New — the centre of the system |
| `zoom_events` | Raw immutable webhook log with a uniqueness key | New |
| `class_disputes` | Parent and student reports, resolution | New |

## 8. Content we still need

The design will fail if it is populated with the content that exists today. This list is the blocker.

| # | Item | Owner | Priority |
|---|---|---|---|
| 1 | Portrait photograph of all 20 instructors, square, consistent lighting | Ops | **Critical** |
| 2 | Instructor bios — 150–250 words each, plus institution, qualification, year | Ops + instructors | **Critical** |
| 3 | 30–60 s recitation audio per Quran instructor | Ops | High |
| 4 | Standard fee schedule, publishable, per course and per session length | Finance | **Critical** |
| 5 | Zakat scholarship eligibility and process, written plainly | Director | **Critical** |
| 6 | 8–12 real testimonials with full name, city, program and outcome, plus written consent | Ops | High |
| 7 | 20–30 photographs of real classes and students (consent obtained) | Ops | High |
| 8 | Where-the-money-goes figures for the transparency page | Finance | High |
| 9 | IRS determination letter PDF | Director | High |
| 10 | Full FAQ — 30–40 question/answer pairs | Ops | High |
| 11 | Course outcome statements rewritten as "what the student will be able to do" | Education | Medium |
| 12 | Curriculum module breakdown per course, with Sabaq/Sabqi/Manzil detail | Education | Medium |
| 13 | Real slot availability data for the schedule picker | Ops | Medium |
| 14 | 6–10 launch blog articles from the existing draft | Ahsan | Medium |
| 15 | Sample report card and sample certificate, designed, for the marketing screenshots | Designer | Medium |
| 16 | Leadership bios and photos | Ops | Medium |
| 17 | Confirmed attendance thresholds from three weeks of live Zoom data | Ahsan | **Critical — blocks 7C** |
| 18 | Zoom account restructure: all classes hosted on the organization account, instructors added as free members | Ops | **Critical — blocks everything in 7C** |
| 19 | Recurring Zoom meeting + meeting ID recorded for every active class | Ops | **Critical** |
| 20 | Wording for the weekly parent confirmation email and the absence notifications | Ops | High |

---

## 9. Quality requirements

### 9.1 Accessibility — WCAG 2.1 AA

- All text 4.5:1; large display 3:1. Test the gold and sienna accents specifically.
- Every interactive element reachable and operable by keyboard, with a visible focus ring that is never removed.
- Logical heading order, one H1 per page, landmarks (`header`, `nav`, `main`, `footer`) present.
- Form inputs have persistent visible labels — placeholder-as-label is not acceptable anywhere.
- Errors are announced via `aria-live` and linked from a summary at the top of the form.
- Modals trap focus and restore it on close; Escape closes.
- Icon-only buttons carry `aria-label`.
- Target size 44 × 44 px minimum.
- `prefers-reduced-motion` honoured everywhere.
- Designer deliverable: a focus-state and keyboard-order annotation for every interactive component.

### 9.2 Localisation and RTL

- English is the primary language. Structure copy so Bengali and Arabic can be added without redesign.
- Arabic content blocks are `dir="rtl"` and must be tested inside otherwise-LTR pages.
- Dates displayed as `30 Aug 2026` (never `08/30/26`, which is ambiguous to our Canadian users). Hijri date shown alongside where it is meaningful.
- All times display with their zone label. Never render a bare time.
- Currency always `USD $`; note where CAD is accepted.
- Name fields must accept Unicode and long names — several of our instructors have four-part names.

### 9.3 Performance budgets

| Metric | Target |
|---|---|
| LCP | < 2.0 s (mobile, 4G) |
| CLS | < 0.05 |
| INP | < 150 ms |
| JS on a marketing route | < 180 KB gzipped |
| Fonts | 2 families, 4 weights, `font-display: swap`, self-hosted, subset |
| Images | AVIF/WebP, explicit dimensions, lazy below the fold |

CLS is called out specifically because the current site has a documented CLS defect caused by cached HTML being served before WordPress hooks fire. In Next.js this class of bug disappears — but only if the designer specifies fixed dimensions for every media element and avoids layout that depends on late-loading content.

---

## 10. Handoff

### 10.1 What the designer delivers

1. **Direction presentation** — one hero and one interior page at high fidelity in the chosen direction, before full production.
2. **Design tokens** — colour, type, spacing, radius, shadow, motion, as a Figma variable collection matching the names in Section 5.
3. **Component library** — every component in Section 6.3, all states, desktop and mobile.
4. **Page designs** — every route in Section 7, at `375 px` and `1440 px` minimum. `768 px` where the layout genuinely changes.
5. **Prototype** — clickable, covering the enrolment funnel end to end and the donation widget.
6. **Annotations** — spacing, behaviour, focus order, empty/error/loading states.
7. **Assets** — SVG icon set, arch and girih source shapes, logo lockups, favicon and OG image templates.

### 10.2 Build order

| Phase | Scope |
|---|---|
| 0 | **Runs in parallel, no design needed.** Zoom account restructure, webhook receiver, three weeks of silent data collection to calibrate thresholds. Nothing in the design depends on it, but everything in Phase 4 and 5 depends on its output. |
| 1 | Design system + Home + Courses + Course detail + Instructors + Instructor profile |
| 2 | Enrolment funnel + confirmation + scholarship |
| 3 | Donate + Transparency + Impact |
| 4 | Portal: auth + student + parent, **including `JoinClassButton`, `OverlapBar` and the six attendance states** |
| 5 | Portal: teacher class record + admin live board + flags + reports |
| 6 | Blog, FAQ, Teach with us, legal, seasonal landing pages |

The attendance components in Phase 4 are the highest-risk design work in the project, because they are used many times a day by people who are not sitting at a desk. Prototype them first.

### 10.3 Open questions for the client

1. Do we publish standard fees publicly, or keep them behind the enrolment flow? *(Recommendation: publish. Every competitor does, and hiding it costs us qualified traffic.)*
2. Is `Sabaq / Sabqi / Manzil` shown to parents in those terms, or translated? *(Recommendation: keep the terms and gloss them once — they signal that we teach Hifz properly.)*
3. Does the trial class stay free and unlimited, or become one session?
4. Is 70% of scheduled duration the right threshold for Present? *(Set from Phase 0 data, not now. The badge component must survive the number changing.)*
5. Does the portal replace the WhatsApp and Telegram workflow, or run alongside it? *(This affects notification design significantly.)*
6. Do we launch Bengali as a second locale in v1 or v2?
7. Does the parent see exact minute figures, or only present/absent? *(Exact minutes build trust but invite disputes over a two-minute shortfall.)*
8. Who may approve a correction — admin only, or the education secretary too?
9. Does the donation tab stay inside the student and teacher portals? *(Donors are in the US and Canada; asking scholarship students to donate inside their own learning portal is worth reconsidering. A link to the public donate page may serve better.)*
10. How long are raw Zoom events retained? *(Twelve months suggested.)*
# iqu-web-platform
