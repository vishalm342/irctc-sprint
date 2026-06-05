# IRCTC Problem Discovery — Part A

## Summary
- Total problems documented: 6 (3 given + 3 self-discovered)
- Platform explored: `irctc.co.in` live on 05 June 2026
- Devices used: Desktop browser session
- Evidence captured: Login overlay, booking search screen, and broken page rendering screenshots

***

## Problem 1: Tatkal Booking Crashes at 10:00 AM [Given]

**What is broken:**  
The IRCTC Tatkal booking flow becomes unresponsive or fails at the exact moment Tatkal inventory opens. Users who have already prepared passenger details often hit a spinner, gateway error, forced logout, or session reset before payment can complete.

**Affected users:**  
Time-sensitive Tatkal users, especially office workers, students, and long-distance travelers booking for next-day travel. This also disproportionately affects users in Tier 2 and Tier 3 cities who depend on rail over air travel and cannot easily absorb a missed booking window.

**Frequency:**  
Daily during the Tatkal release window. Highest severity is concentrated between 9:58 AM and 10:05 AM, when concurrent demand spikes.

**Current flow — step by step:**
1. User opens IRCTC around 9:50 AM and signs in before Tatkal opens.
2. User searches a route, picks travel date, and selects a train with Tatkal quota.
3. User enters passenger details in advance or uses saved passenger data.
4. User waits on the final booking screen and clicks **Book Now** close to 10:00 AM.
5. At or just after 10:00 AM, the page freezes or shows a long spinner without queue status.
6. User receives an HTTP 502, timeout, CAPTCHA reset, or forced logout.
7. User refreshes, signs in again, and finds the Tatkal quota already exhausted or moved to waitlist.
8. User is unsure whether payment was attempted, leading to panic checks of bank balance, UPI history, or SMS inbox.

**Where exactly it breaks:**  
The failure happens at Steps 5–7. The booking system cannot gracefully absorb the demand spike, and the UI gives no queue, retry, or recovery state, so repeated user clicks amplify server pressure.

***

## Problem 2: Search Filters Do Not Work Reliably [Given]

**What is broken:**  
The train results page exposes filters for class, quota, availability, and time, but those filters do not consistently match the trains finally shown. On reload or back-navigation, filter state often resets and users must manually reconstruct their search.

**Affected users:**  
Frequent rail travelers, first-time users, senior citizens, and users comparing many train options in one session. Users with accessibility needs are especially affected because they rely on filtering rather than scanning dense result lists.

**Frequency:**  
Intermittent but common. The failure pattern is stronger during busier hours and on routes with many train options.

**Current flow — step by step:**
1. User enters source, destination, and date, then runs a train search.
2. IRCTC displays a long list of trains with multiple classes and availability states.
3. User applies filters such as **Sleeper**, **Available**, or a specific quota.
4. The page reloads and appears to apply the filter.
5. Some results still show waitlist or mismatched classes despite the filter choice.
6. User opens a train to verify details.
7. On returning to the list, the filter state is missing or partially reset.
8. User repeats the same filtering work or abandons filters and scans manually.

**Where exactly it breaks:**  
The breakdown is at Steps 4–7. Filter state and result state are not consistently preserved together, which makes the interface appear unreliable even when some data is technically available.

***

## Problem 3: Seat Selection Resets Randomly [Given]

**What is broken:**  
Users can choose a specific berth in the seat map, but the chosen seat is not always carried forward into the passenger details or confirmation state. The system may revert to **Auto** or assign a different berth.

**Affected users:**  
Families traveling together, elderly passengers who need lower berths, women traveling alone who prefer certain positions, and passengers with medical or mobility constraints.

**Frequency:**  
Intermittent across booking sessions, with higher risk on mobile and in slower sessions where page transitions trigger more re-renders.

**Current flow — step by step:**
1. User selects a train, class, and quota, then proceeds to seat selection.
2. The seat map loads with berth availability indicators.
3. User selects a specific berth, such as a lower berth for an elderly traveler.
4. The UI highlights the berth as selected.
5. User clicks **Proceed** to move forward.
6. The next page shows **Auto** or a different berth instead of the chosen one.
7. User goes back to correct it, but the seat map may reload and the previous berth may now appear unavailable.
8. User either accepts the wrong berth or restarts the booking attempt.

**Where exactly it breaks:**  
The failure happens between Steps 4 and 6. The seat-selection state is not reliably persisted across the transition from the seat map component to the downstream passenger form.

***

## Problem 4: Login Overlay Blocks Booking Context and Creates a Dead End [Self-discovered]

**How I found it:**  
While exploring the home booking screen and trying to sign in before searching trains, the login action opened a blocking modal over the booking widget.

**Screenshot or description:**  
See `assets/screenshots/problem-4-login-overlay-blocks-booking.jpg`. The login form appears inside a large white overlay with a wide blank panel on the right, dimming the entire train booking screen behind it.

**What is broken:**  
The login interaction is implemented as a heavy blocking overlay that hides the booking task underneath and gives no contextual reassurance about what will happen after sign-in. The right half of the modal is empty space, so the design consumes attention without helping the user complete the task.

**Affected users:**  
Returning users who sign in before searching, Tatkal users preparing in advance, and users on smaller or slower devices where modal transitions feel more disruptive. This especially hurts users who need to cross-check route, quota, or date while logging in.

**Frequency:**  
Observed directly during homepage login entry. The blocking behavior appears to be the default login pattern whenever sign-in is launched from the landing page.

**Current flow — step by step:**
1. User opens the IRCTC landing page to start a booking.
2. User sees the booking widget and decides to sign in first.
3. User clicks **Login / Register** from the top navigation.
4. A modal opens over the page and dims the underlying booking form.
5. The modal shows the login fields on the left but leaves a large blank area on the right.
6. The user can no longer read or interact with the underlying search context while entering credentials.
7. If the user needs to verify route details or return to search preparation, they must close the overlay and restart their attention flow.
8. The sign-in step feels disconnected from the booking task it is supposed to support.

**Where exactly it breaks:**  
The break is at Steps 4–7. Sign-in becomes a separate blocking state instead of a lightweight continuation of booking, increasing cognitive load at the very start of the journey.

***

## Problem 5: Train Search Form Gives Weak Input Feedback and Feels Laggy [Self-discovered]

**How I found it:**  
While inspecting the default booking screen before starting a route search, the search interface felt sparse and offered little guidance or feedback about what had loaded successfully.

**Screenshot or description:**  
See `assets/screenshots/problem-5-search-form-low-feedback.jpg`. The booking module shows empty **From** and **To** inputs, quota and class selectors, and a search CTA, but there is no visible helper state, recent search memory, or reassurance about how station selection behaves.

**What is broken:**  
The search widget exposes core inputs but provides weak feedback about field behavior, especially for first-time or low-confidence users. When the UI is already slow, this lack of visible guidance makes ordinary delay feel like failure because users cannot tell whether station suggestions, quota changes, or class selections have actually registered.

**Affected users:**  
First-time IRCTC users, senior citizens, infrequent travelers, and users on slower connections who depend on visible confirmation before pressing **Search Trains**. Users searching under time pressure are also affected because they may retype or reselect fields unnecessarily.

**Frequency:**  
High. The low-feedback design is always present, while the lag impact becomes more obvious during heavier traffic or slower network conditions.

**Current flow — step by step:**
1. User lands on the booking widget and begins planning a search.
2. User clicks the **From** field to enter a station.
3. The interface gives very little visible guidance about expected station format or selection progress.
4. User fills date, class, and quota while waiting for the interface to feel ready.
5. Because there is limited confirmation feedback, the user hesitates or rechecks whether inputs were correctly captured.
6. User clicks **Search Trains** with low confidence in whether the state is complete.
7. Any small response delay now feels larger because the page did not establish a trustworthy sense of progress earlier.
8. The user experiences the search flow as laggy and mentally fragile even before results appear.

**Where exactly it breaks:**  
The break is at Steps 3–7. The product does not provide enough interaction feedback or microcopy to make a slow system feel dependable, so ordinary latency becomes amplified as UX friction.

***

## Problem 6: Asset/CSS Loading Failures Make Account and PNR Flows Unreadable [Self-discovered]

**How I found it:**  
While navigating account-related and PNR-related states, the page rendered in a visibly broken form where labels, icons, and layout styling did not load correctly.

**Screenshot or description:**  
See `assets/screenshots/problem-6-broken-rendering-account-flow.jpg`. The page shows missing styling, text labels replacing icons, layout collapse, and an unstyled login panel, making key actions difficult to read.

**What is broken:**  
In some account or login-adjacent screens, the interface appears to load without the expected CSS or asset bundle. This turns a transactional workflow into a broken visual state where labels are misaligned, controls are hard to interpret, and trust in the platform drops immediately.

**Affected users:**  
Users checking PNR, opening login/account flows, or managing details during unstable sessions. This is especially severe for low-technical-confidence users who may interpret the screen as a hacked page, scam page, or total system failure.

**Frequency:**  
Intermittent but severe. It may occur under partial page load failures, unstable network conditions, or frontend asset delivery problems, and even one occurrence is high impact because the task becomes nearly unusable.

**Current flow — step by step:**
1. User opens IRCTC and navigates toward a login, account, or PNR-related interaction.
2. The page begins loading normally.
3. Instead of rendering fully styled UI, parts of the page appear in broken layout form.
4. Icons fall back to text labels, spacing collapses, and visual hierarchy disappears.
5. The login or action panel becomes hard to parse because labels and controls no longer align cleanly.
6. User hesitates, unsure whether the site is safe, broken, or still loading.
7. User refreshes, retries, or abandons the task.
8. Trust in the platform drops because a payment-linked government service now looks technically unstable.

**Where exactly it breaks:**  
The failure starts at Steps 3–5. The frontend does not degrade gracefully when assets or styling fail, so the fallback state is effectively unusable for normal travelers.

***

## Screenshot Mapping

| Problem | Suggested screenshot path | Evidence note |
|---------|---------------------------|---------------|
| Problem 4 | `assets/screenshots/problem-4-login-overlay-blocks-booking.jpg` | Login overlay blocks the search form and wastes screen space |
| Problem 5 | `assets/screenshots/problem-5-search-form-low-feedback.jpg` | Booking/search screen feels sparse and offers weak task feedback |
| Problem 6 | `assets/screenshots/problem-6-broken-rendering-account-flow.jpg` | Broken rendering makes login/account or PNR states unreadable |

## Final Review Checklist
- Every problem above includes what is broken, affected users, frequency, current flow, and exact break point.
- The 3 self-discovered problems are distinct from the 3 given problems.
- Screenshot evidence should be added to `assets/screenshots/` in the repository before opening the PR.
- The PR description should include one inline screenshot plus a short summary table of all 6 issues.