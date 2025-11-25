# NetRanks Endpoint Usage Guide

This document ties every backend HTTP endpoint to:

- The backend implementation file
- Its intended purpose
- Whether the current `nr-console-main` frontend calls it (component + page)
- Pages or flows that depend on it, including required call order

Use this as a single source of truth when you need to know **what endpoint to hit on a given screen** or to spot gaps where backend capabilities are not yet wired up in the UI.

---

## 1. Endpoint Reference With Usage Notes

### 1.1 Authentication & Session Management

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| CreateVisitorSession | `GET` and `GET /api/CreateVisitorSession?secret=` | `backend-main/NetRanks/Api/Auth/CreateVisitorSession.cs` | First call returns client IP, second call with encrypted secret returns anonymous visitor token. | Called twice on app bootstrap by `createOnboardingSession()` inside `src/app/components/OnboardingSessionInitializer.tsx`. Runs before any page renders so public flows have a visitor token. |
| CreateMagicLink | `POST /api/CreateMagicLink` | `backend-main/NetRanks/Api/Auth/CreateMagicLink.cs` | Sends passwordless login link, optionally linking visitor session history. | Triggered from `/signin` via `sendMagicLink()` (`src/features/auth/services/authService.ts`). |
| ConsumeMagicLink | `GET /api/ConsumeMagicLink/{id}/{p1}/{p2}` | `backend-main/NetRanks/Api/Auth/ConsumeMagicLink.cs` | Validates magic link, creates a user session token, migrates visitor data. | Executed by `MagicLinkHandler` on `/login/:magicLinkId/:p1/:p2` to mint the console token. |
| GetUser | `GET /api/GetUser` | `backend-main/NetRanks/Api/Auth/GetUser.cs` | Returns user profile, projects and surveys for the console. | `useUser` context fetches once whenever the route starts with `/console` (all console sub-pages). Required before rendering protected routes. |
| Logout | `GET /api/Logout` | `backend-main/NetRanks/Api/Auth/Logout.cs` | Ends the current session. | _Not wired up yet._ Will be needed for a sign-out button in the console header. |

### 1.2 Project & Member Management

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| CreateNewProject | `POST /api/CreateNewProject` | `Api/Project/CreateNewProject.cs` | Creates a project owned by the current user. | _Not used yet._ Console pages for creating/claiming projects need to call this. |
| RenameProject | `PATCH /api/RenameProject/{id}` | `Api/Project/RenameProject.cs` | Updates project name (owner only). | _Not used yet._ Planned for `/console/project/:projectId`. |
| GetMembers | `GET /api/GetMembers/{id}` | `Api/Members/GetMembers.cs` | Lists members for a project. | _Not used yet._ Console “Members” page placeholder. |
| AddMember | `POST /api/AddMember` | `Api/Members/AddMember.cs` | Invites or reactivates a member. | _Not used yet._ |
| DeleteMember | `DELETE /api/DeleteMember/{id}` | `Api/Members/DeleteMember.cs` | Soft-deletes a member. | _Not used yet._ |

### 1.3 Survey Creation & Editing

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| CreateSurvey | `POST /api/CreateSurvey` | `Api/NewSurvey/CreateSurvey.cs` | Authenticated project survey creation. | _Not used yet._ Console “New Survey” workflow should call this once the UI is ready. |
| CreateSurveyFromBrand | `POST /api/CreateSurveyFromBrand` | `Api/Onboarding/CreateSurveyFromBrand.cs` | Visitor flow — creates survey from BrandFetch selection, returns passwords and question set. | Used on `/questions`, `/review-question`, and `/console/review-question` via `fetchBrandQuestions()`. Triggers when user selects a brand. |
| CreateSurveyFromQuery | `POST /api/CreateSurveyFromQuery` | `Api/Onboarding/CreateSurveyFromQuery.cs` | Visitor flow — creates survey from free text question. | Used on `/questions` (when user typed a custom question) and on `/review-question` if no brand is selected. |
| AddQuestion | `POST /api/AddQuestion` | `Api/Questions/AddQuestion.cs` | Adds a question to an existing survey. | `ConsoleQuestionSection` on `/console/review-question` calls `addQuestion()` when user adds a new prompt. |
| DeleteQuestion | `DELETE /api/DeleteQuestion/{id}` | `Api/Questions/DeleteQuestion.cs` | Soft-deletes a survey question. | Same component as above calls `deleteQuestion()`. (Note: current UI passes array index; this will need to send the real question ID.) |
| ChangeSurveySchedule | `PATCH /api/ChangeSurveySchedule/{id}` | `Api/ExistingSurvey/ChangeSurveySchedule.cs` | Updates recurrence interval and NextRunAt. | _Not used yet._ Intended for console scheduling controls. |
| GetSurvey | `GET /api/GetSurvey/{id}` | `Api/ExistingSurvey/GetSurvey.cs` | Returns survey configuration + passwords for project members. | _Not used yet._ Required when console pages show survey metadata. |
| GetSurveyDashboard | `GET|POST /api/GetSurveyDashboard/{id}` | `Api/ExistingSurvey/GetSurveyDashboard.cs` | Authenticated analytics (filters allowed). | _Not used yet._ Console dashboards should call this instead of the visitor dashboard once implemented. |
| GetDashboardFilterFields | `GET /api/GetDashboardFilterFields/{id}` | `Api/ExistingSurvey/GetSurveyDashboardFilterFields.cs` | Lists permissible filter values. | _Not used yet._ |

### 1.4 Survey Execution & Public Dashboards

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| StartSurvey | `GET /api/StartSurvey/{id}` _(visitor auth)_ and `POST /api/StartSurvey/{id}` _(filter specific question indices)_ | `Api/Surveying/StartSurvey.cs` | Launches a survey run, enqueues answer jobs, returns run ID. | `/questions` page calls `startSurvey()` when user hits “Start AI survey”. If the user removed questions, the UI first tries the POST variant (with `questionIndices`), then falls back to GET. |
| GetSurveyRun | `GET /api/GetSurveyRun/{surveyRunId}/{p1}/{p2}` | `Api/Surveying/GetSurveyRun.cs` | Public progress endpoint with brand tallies (auth via survey passwords). | `/brand-rank/survey/:surveyRunId/:p1/:p2` polls this every ~1.5s to animate progress. |
| GetSurveyRunDashboard | `GET /api/GetSurveyRunDashboard/{surveyRunId}/{p1}/{p2}` | `Api/Surveying/GetSurveyRunDashboard.cs` | Visitor dashboard snapshot for a single run. | `/dashboard/:surveyRunId/:p1/:p2` loads this once. |

### 1.5 Question Generation & AI Helpers

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| GenerateQuestionsFromBrand | `POST /api/GenerateQuestionsFromBrand` | `Api/NewSurvey/GenerateQuestionsFromBrand.cs` | Standalone AI helper that returns brand description + questions (no survey creation). | **Not called directly**; `CreateSurveyFromBrand` wraps it server-side. |
| GenerateQuestionsFromQuery | `POST /api/GenerateQuestionsFromQuery` | `Api/NewSurvey/GenerateQuestionsFromQuery.cs` | Same as above for arbitrary prompts. | **Not called directly**; used inside `CreateSurveyFromQuery`. |
| MaybeGenerateQuestionFromWebsite | Internal helper | `Api/NewSurvey/MaybeGenerateQuestionFromWebsite.cs` | Not an endpoint (used internally). | N/A. |

### 1.6 Automation & Console Extras

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| CreateAutomatedSurveyDashboard | `POST /api/CreateAutomatedSurveyDashboard` | `Api/Automation/CreateAutomatedSurveyDashboard.cs` | Creates a survey & immediately starts a run for demo dashboards. | _Not used yet._ Could power a “generate sample dashboard” CTA. |

### 1.7 Billing & Payments

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| CreateSetupIntent | `GET /api/CreateSetupIntent/{projectId}` | `Api/Billing/CreateSetupIntent.cs` | Creates Stripe setup intent for a project owner. | _Not used yet._ Needed when `/console/settings` gets billing. |
| SetPaymentMethod | `POST /api/SetPaymentMethod/{projectId}` | `Api/Billing/SetPaymentMethod.cs` | Persists Stripe payment method to project. | _Not used yet._ |
| GetPaymentMethod | `GET /api/GetPaymentMethod/{projectId}` | `Api/Billing/GetPaymentMethod.cs` | Returns masked card info. | _Not used yet._ |

### 1.8 Content & Marketing

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| GetBlogPosts | `GET /api/GetBlogPosts` | `Api/Blog/GetBlogPosts.cs` | Public blog list. | _Not used yet._ Marketing site could consume this. |
| GetBlogPost | `GET /api/GetBlogPost/{id}` | `Api/Blog/GetBlogPost.cs` | Visitor-auth gated full article. | _Not used yet._ |
| JoinWaitlist | `GET /api/JoinWaitlist?email=` | `Api/Onboarding/JoinWaitlist.cs` | Adds an email to waitlist, redirects. | _Not used yet._ Would be tied to marketing CTA. |

### 1.9 Diagnostics / Samples

| Endpoint | Method & Route | Backend File | Purpose | Frontend Usage |
| --- | --- | --- | --- | --- |
| HelloWorlds | `GET|POST /api/HelloWorlds` | `Api/_HelloWorld/HelloWorlds.cs` | Sample query with delay toggles. | Developer only; not surfaced. |
| RunHelloWorldHttp | `GET|POST /api/RunHelloWorldHttp` | `Api/_HelloWorld/RunHelloWorldHttp.cs` | Simulates a job with fixed 5s duration. | Developer only. |
| BatchHelloWorlds | `GET|POST /api/BatchHelloWorlds` | `Api/_HelloWorld/BatchHelloWorlds.cs` | Stress test that enqueues run jobs. | Developer only. |

---

## 2. Page-by-Page Call Flows

The `OnboardingSessionInitializer` runs on **every page load**. Unless a visitor already has a stored token, it will call `CreateVisitorSession` twice (plain + encrypted) to obtain one and then persists it to `localStorage`.

### `/` — Brand Rank Landing
- **Calls:** none (besides the global onboarding call). Brand search uses the public BrandFetch API directly.
- **Follow-up:** When the user selects a brand or submits a custom prompt, they are navigated to `/questions`, where the survey creation endpoints fire.

### `/questions`
1. **Survey generation**
   - Brand selected → `CreateSurveyFromBrand`.
   - Free-text query (URL `?question=` or inline entry) → `CreateSurveyFromQuery`.
2. **Optional editing** (if user hides questions) happens locally.
3. **Run launch**
   - Primary call: `StartSurvey` (POST with `questionIndices` if any questions were removed, else GET).
4. **Navigation** to `/brand-rank/survey/{runId}/{p1}/{p2}` with returned IDs and passwords.

### `/brand-rank/survey/:surveyRunId/:p1/:p2`
- Polls `GetSurveyRun` every ~1.5 seconds until `Finished == Total`.
- When complete, user can jump to `/dashboard/:runId/:p1/:p2`.

### `/dashboard/:surveyRunId/:p1/:p2`
- Single call to `GetSurveyRunDashboard` to load visitor-facing analytics.
- No further backend traffic (all charts are based on that payload).

### `/signin`
- Form submit → `CreateMagicLink` with email, locale, and (if present) visitor session token.
- Success route → `/magic-link-sent`.

### `/magic-link-sent`
- Static confirmation, no API calls.

### `/login/:magicLinkId/:p1/:p2`
- Mount effect triggers `ConsumeMagicLink`.
- If success → console token stored, redirect to `/console`.

### `/console/*` protected routes
- Before any console child renders, the `UserProvider` auto-fetches `GetUser`.
- Currently no other endpoints are called from dashboard/alerts/settings placeholders.

#### `/console/review-question`
- Reuses the same survey data as `/questions` (brand/query endpoints).
- Adds editing capabilities that hit:
  - `AddQuestion` when the user adds a prompt.
  - `DeleteQuestion` when removing prompts (bug: currently sends index, should send question ID).
- Future steps (not yet wired) should call `ChangeSurveySchedule`, `CreateSurvey`, etc.

#### `/console/members`, `/console/project/:projectId`, `/console/settings`, `/console/support`, `/console/alerts`
- No backend traffic yet. These pages will eventually call the member, project, billing, and support endpoints listed above.

### `/magic-link-sent`, `/console/support`, other static screens
- No backend calls besides the global onboarding run (if token missing).

---

## 3. Implementation Notes & Gaps

1. **Visitor onboarding happens automatically**. If you build a new public page, you usually do _not_ need to call `CreateVisitorSession` manually—include `<OnboardingSessionInitializer />` (already part of `AppProviders`).
2. **Survey passwords (`PasswordOne`, `PasswordTwo`)** returned by the survey creation endpoints must be preserved; every public dashboard/run endpoint requires them.
3. **Console-ready endpoints exist but are unused.** Anything dealing with authenticated projects (CreateSurvey, CreateNewProject, billing, member management) is functional server-side. When implementing console flows, rely on those instead of the visitor endpoints.
4. **Add/Delete question calls need IDs.** The UI currently passes the question’s array index to `api/DeleteQuestion/{id}`. The API expects the database ID. Fixing this will require storing the returned IDs from survey creation.
5. **Dashboard filtering & scheduling** – endpoints (`GetSurveyDashboard`, `GetDashboardFilterFields`, `ChangeSurveySchedule`) are live but unused. Migrating the console dashboard to them will unlock richer analytics and authenticated filters.
6. **Automation endpoint** – `CreateAutomatedSurveyDashboard` can be leveraged to produce on-demand demos from any CTA without exposing project data.

---

## 4. Next Steps

1. Wire console project/billing/member screens to their endpoints.
2. Update question editing UI to use real question IDs for add/delete operations.
3. Consider switching console dashboards from the visitor `GetSurveyRunDashboard` to authenticated `GetSurveyDashboard` for better data.
4. Expose logout and waitlist endpoints in the UI.

With this guide, you can quickly identify which backend function backs a UI action and ensure each page calls the right API sequence. Let’s keep this file updated whenever new endpoints or pages are introduced.

