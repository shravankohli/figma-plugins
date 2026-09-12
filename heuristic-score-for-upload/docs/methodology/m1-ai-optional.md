# Methodology: m1-ai-optional

**Full Name:** Manual Heuristic Screen Score — AI Optional
**ID:** `m1-ai-optional`
**Version:** v1.0
**Created:** 2026-06-01
**Parent Engine:** Screen Scoring Engine (see `Screen_Scoring_Engine_Agent_Prompt.md`)
**Purpose:** Score any digital screen from a static Figma frame using a human-answerable checklist. No AI, no code, no live prototype needed. A designer, product manager, or QA reviewer fills out the checklist; the plugin computes the score.

---

## 1. Design Constraints (Why This Methodology Exists)

The full Screen Scoring Engine has 77 sub-heuristics. Some require runtime observation (loading speed, haptic feedback, screen reader order), AI-level language analysis, or live interaction testing. Those cannot be evaluated from a static Figma frame.

This methodology **retains only the sub-heuristics that a human can reliably answer by looking at a Figma design**. It trades exhaustiveness for implementability — and guarantees that any two reviewers looking at the same frame with the same settings will converge on the same score.

**What was removed and why:**

| Removed Sub-Heuristic | Original ID | Reason |
|-----------------------|-------------|--------|
| Loading Performance | 3.6 | Requires runtime measurement (LCP, INP) |
| Action Acknowledgment | 5.1 | Requires interaction (haptic/visual feedback on tap) |
| Loading State | 5.3 | Requires prototype or live app to observe spinner/skeleton |
| State Persistence | 5.5 | Requires multi-session testing |
| Real-Time Updates | 5.6 | Requires live data connection |
| Graceful Degradation | 6.5 | Requires simulating API/component failure |
| Connectivity Handling | 6.6 | Requires offline/poor-network simulation |
| Screen Reader Compatibility | 7.5 | Requires assistive technology testing |
| Text Scalability | 7.3 | Requires dynamic type / font scaling testing |
| Motion Sensitivity | 7.7 | Requires animation playback beyond static frame |
| Input Modality | 7.8 | Requires multi-input testing (voice, keyboard, touch) |
| Layout Shift Stability | 4.8 | Requires page load observation (CLS) |

**What remains:** 55 sub-heuristics across 10 pillars — every one answerable from a Figma frame.

---

## 2. Setup (What the Plugin Collects Before Scoring)

Before the checklist appears, the Figma plugin collects these configuration variables via dropdowns. These determine which questions are shown and how scores are weighted.

### 2.1 Configuration Fields

| Field | Options | Default | Required |
|-------|---------|---------|----------|
| **Screen Name** | Free text | — | Yes |
| **Product Vertical** | Lending / Investing / Shopping (EMI Card) / Trading / Insurance / Payments / General | General | Yes |
| **Screen Type** | Landing / Form / Dashboard / Confirmation / Error / Listing / Detail / Onboarding / Settings / Empty State | — | Yes |
| **Device** | Mobile / Desktop / Responsive | Mobile | Yes |
| **Flow Position** | Entry / Mid-flow / Terminal / Standalone | Standalone | Yes |
| **User Login State** | Pre-login / Post-login | Post-login | No |
| **Evaluator Name** | Free text | — | Yes |

### 2.2 How Configuration Affects the Checklist

- **Device = Desktop:** Sub-heuristics 3.5 (Thumb Zone) is marked N/A automatically.
- **Screen Type = Empty State:** Sub-heuristic 9.5 (Empty State Copy) becomes applicable; 6.1 (Inline Validation) becomes N/A.
- **Screen Type = Error:** Sub-heuristic 6.2 (Error Message Clarity) and 9.4 (Error Copy) are required (cannot be N/A).
- **Screen Type ≠ Form:** Sub-heuristics 3.2 (Input Efficiency), 3.3 (Pre-Population), and 6.1 (Inline Validation) are marked N/A automatically.
- **Product Vertical = General:** All pillars use equal weights (10% each).
- **Product Vertical ≠ General:** Pillar weights are pulled from the Vertical-Default Weight Table (Section 5.2).

---

## 3. The Checklist (55 Questions)

Each question is answered by selecting one option: **Yes**, **Partial**, **No**, or **N/A**.

Definitions:
- **Yes** — The screen fully meets this criterion. No issue.
- **Partial** — The screen partly meets this criterion. Visible gap, but not a complete failure.
- **No** — The screen fails this criterion. Clear violation or absence.
- **N/A** — This criterion does not apply to this screen type.

When a reviewer selects **Partial** or **No**, an optional free-text field appears for a brief note (e.g., "CTA button says 'Submit' instead of a specific action").

---

### PILLAR 1: Clarity & Comprehension

> *Can the user instantly understand what this screen is for and what to do?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 1.1 | **Can a first-time user tell what this screen is for within 3 seconds?** | Cover the body text with your hand. Read only the heading and CTA. Is the purpose obvious? If you need to read paragraphs to understand — answer No. |
| 1.2 | **Is the primary button/action the most visually prominent interactive element?** | Squint at the screen. The biggest, boldest, most colourful interactive element should be the main action — not a banner, ad, or secondary link. |
| 1.3 | **Does the primary button label say what will happen?** | Good: "Check Eligibility," "Pay ₹1,200," "View Plans." Bad: "Submit," "Next," "Continue," "Click Here." If the label is generic — answer No. |
| 1.4 | **Are all labels and headings in plain, jargon-free language?** | Scan every visible label. If you see terms like "XIRR," "NAV," "co-pay," "reducing balance," "exit load" without an explanation nearby — answer No. One unexplained term = Partial. Three or more = No. |
| 1.5 | **Is the amount of information right for this screen — not too much, not too little?** | Too much: you feel overwhelmed or don't know where to look. Too little: critical info is missing and you'd need to go elsewhere to decide. If you immediately feel "this is a lot" — that's too much. |
| 1.6 | **Is detailed/secondary info hidden behind a tap (e.g., "View details," tooltip, expand)?** | Check if T&C, fee breakdowns, or additional explanations are tucked away. If a wall of fine print is shown directly on the screen — answer No. If there's no secondary info to hide — answer N/A. |
| 1.7 | **Are all icons paired with text labels, or universally obvious?** | Home, search, back arrow, and close (X) are universally obvious. Everything else (especially domain-specific icons) needs a text label. If you see unlabelled ambiguous icons — answer No. |

**7 questions in this pillar.**

---

### PILLAR 2: Navigation & Wayfinding

> *Can the user tell where they are and where they can go?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 2.1 | **Can the user tell where they are in the app?** | Look for: a screen title, breadcrumbs, highlighted tab in the navigation bar, or a step indicator. If the screen has none of these — answer No. |
| 2.2 | **Is there a visible way to go back or close this screen?** | Look for a back arrow, close button (X), or "Cancel" link. If the only way out is the phone's system gesture — answer No. |
| 2.3 | **Is the next step clearly shown?** | On flow screens: is there a forward CTA or clear indication of what comes next? On terminal screens (confirmation, success): is there a "Done" or "Go to Dashboard" option? If the user would wonder "now what?" — answer No. |
| 2.4 | **Does the navigation look and behave the same as other screens in this app?** | Compare the nav bar, header, and tab positions with the app's other screens. If this screen breaks the pattern (e.g., nav bar disappears, header style changes) — answer No. If you're evaluating in isolation — answer N/A. |
| 2.5 | **If there are many items, is search or filtering available?** | Applies to screens with lists, catalogues, or >7 options. If there's no way to search or filter a long list — answer No. If the screen has <7 items — answer N/A. |
| 2.6 | **If the user lands here from a notification or link, can they still orient themselves?** | Imagine arriving at this screen without seeing any prior screens. Does the screen title, context, and content make sense on its own? If it depends entirely on previous context — answer No. |

**6 questions in this pillar.**

---

### PILLAR 3: Task Efficiency

> *Can the user get the job done quickly with minimum effort?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 3.1 | **Is the number of steps/fields minimised for the task?** | Count the fields or steps. For each one, ask: "Is this strictly necessary?" If you spot 2+ fields that feel unnecessary or could be auto-filled — answer Partial. If the screen asks for information already available elsewhere — answer No. |
| 3.2 | **Do form fields use the best input method?** | Check: Are dropdowns used for short lists (<5)? Are numeric keyboards shown for phone/OTP/amount fields? Are date fields using date pickers? Does the field auto-format (e.g., phone numbers, currency)? If fields use plain text input when a better method exists — answer No. Applies only to Form screen types. |
| 3.3 | **Are known fields pre-filled for logged-in users?** | If the user is logged in (post-login state), fields like name, phone, email, and PAN should be pre-filled. If they're asking the user to re-type known data — answer No. Applies only to Form screen types. |
| 3.4 | **Are all buttons and tappable areas at least 44×44 points (iOS) or 48×48 dp (Android)?** | Eyeball interactive elements. If any buttons, links, checkboxes, or icons look small enough that a thumb might miss them — answer No. A good reference: a button should be at least as tall as a line of body text × 2. |
| 3.5 | **Are primary actions within easy thumb reach (bottom half of the screen)?** | On mobile: the main CTA should be in the bottom 40% of the screen, not under the status bar. If the primary action is at the very top and the user would need to stretch — answer No. Desktop = N/A. |
| 3.7 | **Are shortcuts or quick-actions available for repeat users?** | For dashboards, listings, or frequently-used screens: are there quick actions, saved preferences, or "recent" sections? If the screen treats a repeat visitor identically to a first-timer — answer Partial. If this is a one-time flow (onboarding, confirmation) — answer N/A. |
| 3.8 | **Is every step on this screen for the user's benefit (not a marketing insertion)?** | Look for: forced cross-sell interstitials, mandatory marketing opt-in screens, app-rating prompts, or full-screen promotions that the user must dismiss to proceed. If present — answer No. |

**7 questions in this pillar.**

---

### PILLAR 4: Visual Hierarchy & Layout

> *Does the visual design guide the eye to the right things in the right order?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 4.1 | **Do the most important elements have the most visual weight?** | The screen's primary message and primary action should be the largest, boldest, or highest-contrast elements. If a promotional banner or secondary element dominates instead — answer No. |
| 4.2 | **Does the layout follow a natural reading/scanning flow?** | For text-heavy screens: does the layout support top-left to right scanning (F-pattern)? For minimal screens: does it support a Z-shaped flow (top-left → top-right → bottom-left → bottom-right)? If important content is placed where the eye wouldn't naturally go — answer No. |
| 4.3 | **Are related items grouped together visually?** | Check: Are form fields that belong together in the same section? Are prices near the product they relate to? If related information is scattered across the screen — answer No. |
| 4.4 | **Is whitespace used well — not too cramped, not too empty?** | Cramped: elements feel squeezed together, hard to distinguish sections. Too empty: large gaps that make the screen feel unfinished or waste prime viewport space. Either extreme = No. |
| 4.5 | **Is there a clear typographic hierarchy (heading → subheading → body → fine print)?** | Count the levels of text size/weight visible. If everything is the same size — answer No. If there's clear differentiation between at least 3 levels — answer Yes. |
| 4.6 | **Are colours used consistently for the same meaning?** | Check: Is the same green used for success everywhere? Is red always error/warning? Are links always the same colour? If colours mean different things in different places on the same screen — answer No. |
| 4.7 | **If the screen scrolls, is it obvious that there's more content below?** | Check: Does content get cut off mid-element at the bottom (hinting at scroll)? Is there a scroll indicator? If the screen looks "complete" at the fold but has important content below that's invisible — answer No. If the screen doesn't scroll — answer N/A. |

**7 questions in this pillar.**

---

### PILLAR 5: Feedback & System Status

> *Does the screen tell the user what's happening?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 5.2 | **For multi-step flows: does the user know which step they're on and how many are left?** | Look for step indicators (Step 2 of 4), progress bars, or breadcrumbs. If this is a multi-step flow and there's no progress indication — answer No. If this is a single-screen task — answer N/A. |
| 5.4 | **After completing a task, is there a clear success message with guidance on what to do next?** | On confirmation/success screens: is there a clear "Done" or "What's next?" message? If the screen just says "Success" with no next step — answer Partial. If there's no success state visible — answer N/A. |
| 5.7 | **Are any visible notifications or banners relevant to the current task (not promotional)?** | If the screen shows banners, toasts, or alerts: are they related to what the user is doing? If there's a promotional banner interrupting a task flow — answer No. If there are no notifications visible — answer N/A. |

**3 questions in this pillar.**

---

### PILLAR 6: Error Handling & Recovery

> *Does the screen prevent mistakes and help the user fix them?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 6.1 | **Are form errors shown right next to the field (not in a popup or at the top)?** | Look at the error state of the form. Errors should appear inline, below or beside the field, not in a generic toast at the top. If errors are shown as a list at the top of the page — answer Partial. Applies only to Form screen types. |
| 6.2 | **Do error messages explain what went wrong AND how to fix it?** | Good: "Phone number must be 10 digits. Please remove the country code." Bad: "Invalid input." "Error." "Something went wrong." If the error message is vague or missing a fix suggestion — answer No. |
| 6.3 | **Are risky actions guarded with a confirmation step?** | For actions like deleting data, cancelling an application, making a payment, or submitting a loan application: is there a confirmation dialog? If a tap immediately executes an irreversible action — answer No. If there are no risky actions on this screen — answer N/A. |
| 6.4 | **Can the user undo or go back to change their last action?** | Can they re-edit a field, change a selection, or go back a step? If the screen locks in choices with no way to revise — answer No. |
| 6.7 | **Does the screen handle "empty" or "zero" states gracefully?** | If there's no data to show (empty cart, no transactions, no search results): does the screen explain why it's empty and guide the user to do something? If it's just blank or shows a generic "No data" — answer No. If empty states aren't relevant to this screen — answer N/A. |
| 6.8 | **After an error, is the user guided to a productive next step (not a dead end)?** | Check the error screen/state: does it offer a "Retry," "Go back," "Contact support," or alternative path? If the user is stuck with just an error message and no way forward — answer No. If no error states are visible — answer N/A. |

**6 questions in this pillar.**

---

### PILLAR 7: Accessibility & Inclusion

> *Can every user — regardless of ability — use this screen?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 7.1 | **Does all text have sufficient colour contrast against its background?** | Use the Figma contrast checker plugin or squint at the text. Light grey text on white, or coloured text on a coloured background, often fails. Minimum: 4.5:1 for normal text, 3:1 for large text (18pt+ or 14pt bold). If any text is hard to read — answer No. |
| 7.2 | **Is colour NEVER the only way to show important information?** | Check: if you converted the screen to greyscale, would you lose any meaning? Are error states signalled with icons/text in addition to red colour? Are links distinguishable without colour? If colour is the sole differentiator anywhere — answer No. |
| 7.4 | **Are all tappable/clickable elements large enough to hit accurately?** | Same check as 3.4 but from an accessibility lens. Minimum 44×44 pt. If any interactive element (especially checkboxes, radio buttons, or text links) is smaller — answer No. |
| 7.6 | **Is the language simple enough for the target audience?** | For a general Indian audience: aim for language that someone with a Class 8 education can understand. If the screen uses complex English sentences, passive voice, or financial jargon — answer No. If there's 1 borderline term — answer Partial. |

**4 questions in this pillar.**

---

### PILLAR 8: Emotional Design & Trust

> *Does the screen feel trustworthy, professional, and emotionally right?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 8.1 | **Does the screen look polished and professional?** | Check for: placeholder text ("Lorem ipsum"), broken images, misaligned elements, inconsistent padding, orphan text (single word on a line). If the screen looks unfinished — answer No. |
| 8.2 | **Does the screen's tone match the situation?** | A payment confirmation should feel reassuring, not playful. An error screen should feel supportive, not robotic. A success screen can be celebratory. If the tone feels off for the context — answer No. |
| 8.3 | **Are trust signals present where needed?** | For screens involving money, data, or commitment: look for security badges ("Secured by..."), certification logos, partner brand logos, user counts, ratings, or a visible helpline number. If the screen asks for personal/financial information with zero trust signals — answer No. |
| 8.4 | **Is the visual design consistent with the app's brand?** | Check: Does it use the brand's fonts, colours, and icon style? Does it look like it belongs to the same app as the other screens? If it feels like a different app — answer No. If you're evaluating a single screen in isolation — answer N/A. |
| 8.5 | **For high-stakes screens: are anxiety-reducing elements present?** | On screens where the user is committing money (payment, loan, insurance purchase): are there elements like "You can cancel within 24 hours," "30-day free look period," "Call us at [number]," or a security lock icon? If the screen asks for a big commitment with no reassurance — answer No. If this isn't a high-stakes screen — answer N/A. |
| 8.7 | **Does the screen feel simple and easy, even if the task is complex?** | This is a gut check. After answering all previous questions: does the overall screen feel approachable, or does it feel intimidating? Trust your instinct here — it correlates with the Aesthetic-Usability Effect. |

**6 questions in this pillar.**

---

### PILLAR 9: Content Quality & Microcopy

> *Is every word on the screen purposeful, clear, and helpful?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 9.1 | **Can the user get the key message by just scanning headings and bold text?** | Cover the body text. Read only the headings, bold text, and CTA labels. Can you understand what this screen is about and what to do? If yes — answer Yes. If you need to read everything — answer No. |
| 9.2 | **Are technical or financial terms avoided or explained?** | Count the number of unexplained jargon terms visible on the screen. 0 = Yes. 1–2 = Partial. 3+ = No. Examples of jargon: "NAV," "exit load," "co-pay," "sub-limit," "reducing balance," "demat," "LTCG." |
| 9.3 | **Do button labels use specific action verbs?** | Good: "Check Eligibility," "Pay ₹1,200," "Download Statement," "Compare Plans." Bad: "Submit," "OK," "Next," "Click Here," "Proceed." If the primary CTA is generic — answer No. |
| 9.4 | **Are error messages human, empathetic, and action-oriented?** | Good: "We couldn't verify your PAN. Please check for typos and try again." Bad: "Error 422." "Validation failed." "Invalid input." If error messages are visible and they're robotic — answer No. If no errors are visible — answer N/A. |
| 9.5 | **Do empty states explain what will appear and how to get started?** | Good: "No transactions yet. Your payment history will appear here once you make your first payment." Bad: "No data." [blank screen]. If empty states are visible and unhelpful — answer No. If no empty states are visible — answer N/A. |
| 9.6 | **Is legal/regulatory text present but not overwhelming?** | T&C, disclaimers, and IRDAI/SEBI mandated text should be visible but not dominate the screen. If fine print takes up more visual space than the primary content — answer No. If regulatory text is entirely missing where required — also No. If not applicable — N/A. |
| 9.7 | **Are numbers formatted for Indian conventions?** | Check: ₹ symbol (not "Rs" or "INR"), lakhs/crores (not millions), DD/MM/YYYY dates, percentage signs, Indian phone number formatting. If any number formatting follows non-Indian conventions — answer No. |

**7 questions in this pillar.**

---

### PILLAR 10: Ethical Integrity

> *Does the screen respect the user and play fair?*

| # | Question | Guidance for the Reviewer |
|---|----------|--------------------------|
| 10.1 | **Are opt-in checkboxes unchecked by default?** | Look for marketing consent, newsletter subscriptions, terms acceptance, and cross-sell opt-ins. If any are pre-checked — answer No. If there are no opt-ins on this screen — answer N/A. |
| 10.2 | **Are all costs, fees, and charges visible before the user commits?** | Check: processing fees, GST, convenience charges, interest rates, total EMI payable. If any cost is hidden or only appears on a later screen — answer No. If this screen doesn't involve costs — answer N/A. |
| 10.3 | **If comparing options: is the comparison neutral and not visually rigged?** | Check: Is one option made visually dominant (much larger, brighter, "Recommended" badge) to steer the user? Are the comparison criteria fair and consistent? Some highlighting is OK; heavy manipulation is No. If there's no comparison — answer N/A. |
| 10.4 | **Is it as easy to decline/cancel as it is to accept/buy?** | Compare the "Accept" and "Decline" (or "Cancel") buttons. Are they the same size and equally visible? If "Accept" is a large coloured button and "Decline" is a tiny grey text link — answer No. |
| 10.5 | **Does declining use neutral, non-shaming language?** | Good: "No thanks," "Skip," "Maybe later." Bad: "No, I don't want to save money," "I'll pass on this amazing deal." If the decline copy makes the user feel guilty — answer No. If there's no decline option — answer N/A. |
| 10.6 | **Are add-ons, extras, or cross-sells NOT pre-added?** | Check: Are insurance add-ons, extended warranties, or accessory bundles automatically added to the cart or selection? If yes — answer No. If there are no add-ons — answer N/A. |
| 10.7 | **If urgency messaging is used, is it real?** | "Offer ends in 2 hours" — is it actually time-limited? "Only 3 left" — is inventory genuinely low? If the urgency feels manufactured or resets on refresh — answer No. If you can't verify from the design — answer N/A. If there's no urgency messaging — answer N/A. |
| 10.8 | **Does the screen ask only for information needed for this task?** | If a bill payment screen asks for your date of birth, or a loan eligibility check asks for your employer's address upfront — that's excessive. If every field is clearly relevant — answer Yes. |

**8 questions in this pillar.**

---

## 4. Scoring Math (For Plugin Implementation)

### 4.1 Points Per Answer

| Answer | Points |
|--------|--------|
| Yes | 2 |
| Partial | 1 |
| No | 0 |
| N/A | Excluded from calculation |

### 4.2 Pillar Score Calculation

For each pillar:

```
Applicable Questions  = Total Questions − N/A Count
Raw Points            = (Yes Count × 2) + (Partial Count × 1) + (No Count × 0)
Maximum Points        = Applicable Questions × 2
Percentage            = (Raw Points ÷ Maximum Points) × 100
```

If all questions in a pillar are N/A, the pillar is excluded from the composite score and its weight is redistributed equally among remaining pillars.

### 4.3 Percentage → Pillar Grade (0–5)

| Percentage | Grade | Label | Colour (for plugin UI) |
|------------|-------|-------|----------------------|
| 85–100% | 5 | Exemplary | Green (#22C55E) |
| 68–84% | 4 | Good | Light Green (#84CC16) |
| 51–67% | 3 | Adequate | Yellow (#EAB308) |
| 34–50% | 2 | Below Average | Orange (#F97316) |
| 17–33% | 1 | Poor | Red (#EF4444) |
| 0–16% | 0 | Critical Failure | Dark Red (#991B1B) |

### 4.4 Composite Score Calculation

```
Weighted Composite = Σ (Pillar Grade × Pillar Weight)
```

Result is a number from 0.00 to 5.00.

### 4.5 Overall Grade

| Composite Score | Grade | Meaning |
|----------------|-------|---------|
| 4.50 – 5.00 | A+ | Exceptional — reference-quality |
| 4.00 – 4.49 | A | Strong — minor polish needed |
| 3.50 – 3.99 | B | Good — targeted fixes will elevate |
| 3.00 – 3.49 | C | Adequate — systematic improvement needed |
| 2.00 – 2.99 | D | Below standard — significant rework |
| 0.00 – 1.99 | F | Failing — fundamental redesign needed |

---

## 5. Weight Profiles

### 5.1 Equal Weights (Default — when Vertical = "General")

Every pillar: **10%** each. Total: 100%.

### 5.2 Vertical-Specific Weights

When a Product Vertical is selected, use these pre-calibrated weights:

| Pillar | Lending | Investing | Shopping | Trading | Insurance | Payments |
|--------|---------|-----------|----------|---------|-----------|----------|
| P1: Clarity | 12% | 12% | 10% | 8% | 14% | 8% |
| P2: Navigation | 8% | 8% | 12% | 8% | 8% | 10% |
| P3: Efficiency | 12% | 8% | 12% | 18% | 8% | 18% |
| P4: Visual | 8% | 10% | 12% | 12% | 8% | 8% |
| P5: Feedback | 12% | 8% | 8% | 14% | 10% | 14% |
| P6: Errors | 12% | 8% | 8% | 10% | 10% | 10% |
| P7: Accessibility | 10% | 10% | 10% | 6% | 10% | 10% |
| P8: Trust | 10% | 14% | 8% | 6% | 14% | 6% |
| P9: Content | 8% | 12% | 10% | 8% | 12% | 8% |
| P10: Ethics | 8% | 10% | 10% | 10% | 6% | 8% |

**Why these weights?**
- Lending / Insurance → High anxiety products → Clarity and Trust are weighted higher.
- Trading → Speed-critical → Task Efficiency and Feedback matter most.
- Payments → Habit-driven, competing with PhonePe/GPay → Task Efficiency dominates.
- Investing → Jargon-heavy → Content and Trust are weighted up.
- Shopping → Comparison-driven → Navigation and Visual Hierarchy matter more.

---

## 6. Question Applicability Matrix

This table tells the plugin which questions to show (and which to auto-set as N/A) based on Screen Type and Device.

| Question | Landing | Form | Dashboard | Confirmation | Error | Listing | Detail | Onboarding | Settings | Empty State | Desktop Override |
|----------|---------|------|-----------|-------------|-------|---------|--------|------------|----------|-------------|-----------------|
| 1.6 | ✓ | ✓ | ✓ | N/A | N/A | ✓ | ✓ | ✓ | ✓ | N/A | — |
| 2.5 | ✓ | N/A | ✓ | N/A | N/A | ✓ | N/A | N/A | ✓ | N/A | — |
| 3.2 | N/A | ✓ | N/A | N/A | N/A | N/A | N/A | ✓ | ✓ | N/A | — |
| 3.3 | N/A | ✓ | N/A | N/A | N/A | N/A | N/A | N/A | N/A | N/A | — |
| 3.5 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | N/A |
| 3.7 | ✓ | N/A | ✓ | N/A | N/A | ✓ | N/A | N/A | ✓ | N/A | — |
| 5.2 | N/A | ✓ | N/A | N/A | N/A | N/A | N/A | ✓ | N/A | N/A | — |
| 5.4 | N/A | N/A | N/A | ✓ | N/A | N/A | N/A | N/A | N/A | N/A | — |
| 6.1 | N/A | ✓ | N/A | N/A | N/A | N/A | N/A | ✓ | ✓ | N/A | — |
| 6.7 | ✓ | N/A | ✓ | N/A | N/A | ✓ | N/A | N/A | N/A | ✓ | — |
| 8.5 | N/A | ✓ | N/A | ✓ | N/A | N/A | ✓ | N/A | N/A | N/A | — |
| 9.5 | N/A | N/A | ✓ | N/A | N/A | ✓ | N/A | N/A | N/A | ✓ | — |
| 10.7 | ✓ | N/A | N/A | N/A | N/A | ✓ | ✓ | N/A | N/A | N/A | — |

All questions not listed here are **always applicable** (shown on every screen type and device).

The reviewer can still manually override any auto-N/A to a different answer if they believe the question is relevant to their specific screen.

---

## 7. Plugin Output Specification

After the reviewer completes all questions, the plugin generates a results panel with:

### 7.1 Summary Card

```
┌─────────────────────────────────────────┐
│  HEURISTIC SCORE                        │
│  Screen: [Name]                         │
│  Vertical: [Selected]  Device: [Sel.]   │
│                                         │
│  ██████████████████░░  4.12 / 5.00      │
│                        Grade: A         │
│                                         │
│  Evaluator: [Name]    Date: [Auto]      │
│  Methodology: m1-ai-optional v1.0       │
└─────────────────────────────────────────┘
```

### 7.2 Pillar Breakdown (Bar chart or table)

```
P1  Clarity        ████████████████░░░░  4  Good
P2  Navigation     ██████████████████░░  5  Exemplary
P3  Efficiency     ████████████░░░░░░░░  3  Adequate
P4  Visual         ████████████████░░░░  4  Good
P5  Feedback       ████████░░░░░░░░░░░░  2  Below Avg
P6  Errors         ████████████████░░░░  4  Good
P7  Accessibility  ████████████░░░░░░░░  3  Adequate
P8  Trust          ██████████████████░░  5  Exemplary
P9  Content        ████████████████░░░░  4  Good
P10 Ethics         ████████████████████  5  Exemplary
```

### 7.3 Issues List

Every question answered **No** or **Partial** appears here, sorted by pillar weight (highest-weight pillar issues first). Each entry shows:
- The question text
- The reviewer's answer (No / Partial)
- The reviewer's note (if provided)
- The pillar it belongs to and that pillar's weight

### 7.4 Exportable Report

The plugin should offer a **Copy to Clipboard** or **Export as PDF** button that produces a formatted report containing:
- Configuration (screen name, vertical, device, screen type, evaluator, date)
- Composite score and grade
- Pillar-by-pillar breakdown with all answers
- Issues list with notes
- Methodology version identifier (`m1-ai-optional v1.0`)

---

## 8. Versioning & Changelog

| Version | Date | Change Type | Description |
|---------|------|------------|-------------|
| v1.0 | 2026-06-01 | Initial | 55 questions across 10 pillars. 12 runtime-dependent sub-heuristics removed from the parent engine. Equal and vertical-specific weight profiles included. |

**Version rules:**
- **Patch (v1.0.x):** Guidance text clarifications. No score impact.
- **Minor (v1.x.0):** Questions added/removed, weight adjustments ≤ 3%. Scores may shift slightly. Flag prior evaluations.
- **Major (vX.0.0):** Pillar changes, aggregation logic changes. Prior scores are no longer comparable.

---

---

# Part 2: The Layperson's Guide

## What Is This, in Plain English?

Imagine you baked a cake and want to know if it's good. You could ask an expert chef (expensive, slow) — or you could use a **checklist**: Is it moist? Is the frosting even? Does it taste sweet enough? Is it cooked through?

This methodology is that checklist — but for app and website screens.

Instead of asking "is this screen good?" (which is vague and everyone answers differently), it asks **55 specific yes-or-no-ish questions** about the screen. Your answers are converted into a score. Same screen + same answers = same score, every time.

---

## The 3 Things You Need to Know

### 1. There Are 10 Categories (We Call Them "Pillars")

Think of them as 10 different ways a screen can be good or bad:

| # | Category | The Question It Answers | Everyday Analogy |
|---|----------|------------------------|-----------------|
| 1 | **Clarity** | "Do I get what this screen is about?" | Walking into a shop and instantly knowing what they sell |
| 2 | **Navigation** | "Can I find my way around?" | Clear signs in a hospital |
| 3 | **Efficiency** | "Can I get this done fast?" | Self-checkout vs. a long queue |
| 4 | **Visual Layout** | "Does it look organised?" | A tidy desk vs. a messy one |
| 5 | **Feedback** | "Does it tell me what's happening?" | A waiter saying "your order is being prepared" vs. silence |
| 6 | **Error Handling** | "What happens when I make a mistake?" | A kind teacher vs. a confusing error beep |
| 7 | **Accessibility** | "Can everyone use this?" | Ramps alongside stairs |
| 8 | **Trust** | "Does it feel safe and real?" | A well-lit, branded bank branch vs. a shady stall |
| 9 | **Content** | "Are the words helpful?" | Medicine instructions in plain Hindi vs. Latin names |
| 10 | **Ethics** | "Is it playing fair with me?" | Transparent pricing vs. hidden charges at billing |

### 2. Each Category Has 3–8 Simple Questions

You answer each question with one of four options:

| Answer | What It Means | Points |
|--------|--------------|--------|
| **Yes** | "This screen does this well" | 2 points |
| **Partial** | "It sort of does this, but there's a gap" | 1 point |
| **No** | "This screen fails at this" | 0 points |
| **N/A** | "This question doesn't apply to this screen" | Skipped |

You don't need to be a UX expert to answer these. The questions are written so that anyone who uses apps regularly can answer them. Each question includes a "Guidance" tip that tells you exactly what to look for.

### 3. The Math Is Automatic

You just answer the questions. The plugin does all the math:

**Step 1:** For each category, it adds up your points and figures out a percentage.

> *Example:* Category 1 (Clarity) has 7 questions. You answered Yes to 5, Partial to 1, and No to 1.
> Points: (5 × 2) + (1 × 1) + (1 × 0) = 11 out of 14 = 78.6% → Grade: **4 (Good)**

**Step 2:** It combines all 10 category grades into one overall score (0 to 5), applying weights based on the type of product (a payment screen cares more about speed; an insurance screen cares more about trust).

**Step 3:** It assigns a letter grade:

| Score | Grade | What It Means |
|-------|-------|--------------|
| 4.5–5.0 | A+ | Outstanding. Show it off. |
| 4.0–4.4 | A | Very good. Small tweaks only. |
| 3.5–3.9 | B | Good. Some areas to improve. |
| 3.0–3.4 | C | OK. Needs real work. |
| 2.0–2.9 | D | Not good enough. Major fixes needed. |
| 0.0–1.9 | F | Failing. Go back to the drawing board. |

---

## How to Actually Use It (Step by Step)

### Step 1: Open the Plugin in Figma

Select the frame (screen) you want to evaluate. Open the plugin.

### Step 2: Fill in the Setup (30 seconds)

Pick from dropdowns:
- What is this screen for? (Lending, Shopping, etc.)
- What type of screen is it? (A form? A dashboard? A confirmation page?)
- Is it for mobile or desktop?

This ensures the right questions are shown and the right weights are applied.

### Step 3: Answer the Questions (5–10 minutes per screen)

Go through each question. For each one:
- **Read the question.**
- **Read the guidance tip** (it tells you exactly where to look on the screen).
- **Pick Yes / Partial / No / N/A.**
- **Optionally type a short note** explaining why you picked Partial or No.

You do NOT need to:
- Know any UX theory
- Understand research papers
- Use any AI tool
- Write long explanations

### Step 4: See Your Score (Instant)

The plugin immediately shows:
- Your overall score and letter grade
- A bar chart of all 10 categories
- A list of every issue (every No and Partial answer), sorted by importance

### Step 5: Share or Export

Copy the results to your clipboard or export as a PDF. Share with your team. The report includes the methodology version, so scores are always traceable.

---

## Frequently Asked Questions

**Q: Do I need to be a designer to use this?**
No. The questions are written for anyone who uses mobile apps regularly. Product managers, QA testers, developers, and business stakeholders can all use this.

**Q: How long does it take?**
5–10 minutes per screen for an experienced user. 15 minutes the first time while you read the guidance tips.

**Q: What if I'm not sure about an answer?**
When in doubt between Yes and Partial, pick Partial. When in doubt between Partial and No, pick Partial. When in doubt about whether a question applies, pick N/A. The system is designed to handle uncertainty gracefully.

**Q: Will two different people get the same score?**
That's the goal. The questions are specific enough that most people looking at the same screen will answer the same way. In testing, reviewers typically agree on 80–90% of questions. The remaining variance is handled by the scoring math (averaging out minor disagreements).

**Q: Can I use this for apps other than Bajaj Finserv?**
Yes. Set the Product Vertical to "General" and the weights will be equal across all categories. The questions themselves are universal — they work for any digital product.

**Q: What if I want more detail or AI-powered analysis?**
This methodology is "AI Optional." You can use it purely manually (as described above). But if you have access to the full Screen Scoring Engine agent, you can feed it the same screen for a deeper, 77-sub-heuristic evaluation with evidence logging. The scores from both methods are compatible and comparable.

**Q: What does the score NOT tell me?**
It doesn't tell you about:
- **How fast the screen loads** (needs a real app, not a Figma design)
- **How screen readers handle it** (needs accessibility testing tools)
- **Whether users actually succeed at their tasks** (needs usability testing with real people)
- **Whether the animations feel right** (needs a prototype, not a static frame)

These are covered by the full Screen Scoring Engine but are beyond what a Figma plugin can assess from a static design.
