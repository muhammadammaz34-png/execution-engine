## 1. Frontend Vision

WorkflowOS is not a chat interface with extra features bolted on. It is an execution surface, and that distinction should shape every decision in this document before a single component gets named.

A chat product's central object is the message. The UI exists to move messages back and forth, and everything else, files, tools, code blocks, is a guest inside that thread. WorkflowOS's central object is the run: a goal, a plan, a set of agents doing work, and a trail of evidence a human can inspect and approve. The goal a user types is closer to a commit message than a chat prompt. It kicks off something, and then the interface's job is to show that something happening, clearly, in real time, with the ability to intervene, not to keep a conversation going.

This has a concrete consequence for layout: the largest, most persistent surface in the application is the Execution View (Section 9), not a message list. A text input exists (the Command Box, Section 6), but it is the ignition, not the steering wheel. Once a run starts, the user's attention moves to a timeline of agents, tool calls, and checkpoints, the same way an engineer's attention moves from a `git commit` command to a CI pipeline view, not back to their own terminal history.

Three things follow from treating this as an execution product rather than a conversation product:

**The UI must make autonomy legible.** A chat window hides its reasoning behind a typing indicator. WorkflowOS shows which agent is active, what it is doing, and what it produced, at every step, because the product's entire value proposition rests on a user trusting work they did not personally perform. This is not a nice-to-have panel; it is the product.

**Human checkpoints are first-class UI states, not modals that interrupt a flow.** The Approval Modal (Section 6) and the `awaiting_approval` status in the Execution View are treated with the same visual weight as "running," because in this product, waiting for a human is not an edge case, it is one of maybe four core states every run passes through.

**Density beats charm.** This is a tool an account manager will have open for hours across a workday, running five or six campaigns in parallel across client workspaces. It should read closer to a flight radar or a CI dashboard than a marketing landing page, once the user is past onboarding. Dark-first is not an aesthetic choice alone, it is the correct default for a tool meant to sit open on a second monitor for long stretches.

---

## 2. Design Principles

| Principle | What it means in practice |
|---|---|
| **Minimal** | Every screen answers one question. The Dashboard answers "what needs my attention." The Execution View answers "what is happening right now." No screen tries to answer both. |
| **Modern** | Contemporary SaaS visual language: generous whitespace at rest, dense information display only where the task demands it (the Execution View, the Logs Viewer), never decorative complexity. |
| **Fast** | Perceived speed is a design requirement, not just an engineering one. Every action that triggers backend work gets an immediate, optimistic UI response, a skeleton, a disabled-with-spinner button state, within 100ms, even though the actual campaign creation (Section 5, Workflow Builder) is asynchronous and can take minutes. |
| **Enterprise Ready** | Multi-workspace switching, role-aware UI (Section 16), audit visibility, and keyboard-first workflows are present from Version 1, not retrofitted once a large customer asks for them. |
| **AI Native** | The interface assumes non-deterministic output as the normal case, not the exception. Every AI-generated artifact carries a visible provenance marker (which agent, when, based on what context) rather than presenting generated content indistinguishably from human-authored content. |
| **Human Oversight** | Irreversible actions (Section 16 of the API design: publishing) are never one click away from a passive viewing state. The Approval Modal is a deliberate, separate interaction, never a checkbox buried in a longer form. |
| **Trust Through Transparency** | The Logs Viewer and Tool Calls panel (Section 9) are not debug tools hidden behind a developer flag. They are core product surfaces available to the Account Manager persona from the PRD, because trust in autonomous execution is built by watching it work correctly, repeatedly, not by a marketing claim. |
| **Explainability** | Every validation failure, every flagged tone deviation, every rejected post names the specific rule it violated and the specific text span, never a generic "content did not pass validation." |
| **Progressive Disclosure** | The Dashboard shows status. The Campaign Detail page shows content. The Execution View shows process. A user never has to see agent-level detail to approve a post, but it is always one click away for the user who wants it. |

---

## 3. Design System

### Typography

A single type family (Inter, or an equivalent geometric sans already in the shadcn/ui default stack) at a restrained set of sizes, since a workflow product lives or dies on scanability, not typographic personality.

| Token | Size / Line height | Use |
|---|---|---|
| `text-display` | 32px / 40px, semibold | Page titles only (Dashboard, Campaign Detail headline) |
| `text-heading` | 20px / 28px, semibold | Section headers within a page |
| `text-subheading` | 16px / 24px, medium | Card titles, table headers |
| `text-body` | 14px / 20px, regular | Default UI text, the size most of the application lives at |
| `text-caption` | 12px / 16px, regular | Timestamps, metadata, helper text |
| `text-mono` | 13px / 20px, monospace | Logs Viewer, tool call payloads, IDs |

### Spacing and grid

4px base unit, scaling `1` through `16` (4px to 64px), matching Tailwind's default scale directly so no custom spacing config is needed. Page content sits on a 12-column grid with a 1280px max width on desktop, collapsing per the breakpoints in Section 13. The Execution View is the one deliberate exception: it uses the full viewport width above 1440px, because the timeline and logs panels both benefit from horizontal room that a centered 1280px column would waste.

### Border radius and elevation

Two radius tokens only: `radius-sm` (6px) for inputs, badges, and buttons, `radius-md` (10px) for cards, modals, and panels. No `radius-lg` or fully rounded (pill) elements except badges and the status dot indicators used throughout the Execution View, to keep the interface reading as a serious tool rather than a consumer app.

Elevation uses shadow, not border, to separate layers in dark mode (borders read poorly against dark backgrounds): `elevation-0` (flat, default page background), `elevation-1` (cards, a subtle 1px inner border plus a soft shadow), `elevation-2` (modals, dropdowns, the Command Palette), `elevation-3` (toasts, reserved for anything that must visually sit above everything else including modals).

### Cards, buttons, inputs, badges, alerts, tables, tags

| Component | Key variants |
|---|---|
| **Card** | Default (elevation-1), Interactive (adds hover elevation-2 and cursor-pointer, used for Workflow Cards and Campaign Cards), Highlighted (used for the currently-active step in the Execution Timeline, a colored left border rather than a full background tint, to stay legible in dark mode) |
| **Button** | Primary (filled, one per screen, reserved for the single most important action), Secondary (outlined), Ghost (text-only, used inside tables and toolbars), Destructive (used only for cancel/delete actions, red-600 in both themes), Icon-only (44px hit target minimum, per Section 14) |
| **Input** | Text, Textarea (the Command Box is a specialized, larger textarea), Select (shadcn/ui `Select`), Combobox (brand and workspace pickers), all sharing one focus ring token so keyboard focus is always visually identical regardless of input type |
| **Badge** | Status badges use a fixed color-to-meaning mapping, never assigned ad hoc per screen: gray (pending), blue (in progress), amber (awaiting approval), green (completed), red (failed), slate (cancelled) |
| **Alert** | Inline (within a card, non-blocking, used for validation warnings), Banner (full-width, page-level, used for account-level issues like an expired LinkedIn token) |
| **Table** | Dense by default (32px row height) given the account manager persona's need to scan many campaigns quickly; a "comfortable" density toggle exists as a user preference but dense is the shipped default |
| **Tag** | Used for brand names and workspace labels, visually distinct from status Badges (rounded-full, neutral color, never carries semantic meaning the way a Badge does) |

### Colors

Dark-first, meaning the dark palette is the design baseline and the light palette is derived from it, not the reverse. A semantic token layer sits between raw color values and components, so a designer never hardcodes a hex value into a component spec:

```
--background        near-black, not pure black (#0A0B0D)
--surface            one step lighter, for cards (#131519)
--surface-raised      for modals/dropdowns (#1B1E24)
--border               low-contrast hairline (#262A31)
--text-primary         near-white, not pure white (#EDEEF0)
--text-secondary        (#9498A2)
--text-tertiary          (#63676F)
--accent                a single brand blue, used sparingly, reserved for
                          primary actions and the active-step indicator
--success / --warning / --danger / --info    fixed, WCAG-checked against
                                                both surface tokens
```

### Light theme

A true light theme, not an inverted dark theme with contrast problems papered over. Light mode swaps the semantic token values (`--background` becomes a warm off-white, not pure white, `--text-primary` becomes near-black) while every component references the same token names, so no component-level styling differs between themes.

### Accessibility baseline

Every color pairing in both themes meets WCAG AA (4.5:1 for body text, 3:1 for large text and UI components) as a design system requirement, not a post-launch audit item. Detailed behavior in Section 14.

---

## 4. Application Layout

The authenticated shell is consistent across every page except Landing, Login, Signup, and 404, which use a minimal unshelled layout described in their own page specs.

```
+----------------------------------------------------------------------------+
| [Logo] [Workspace Switcher v]        [Command Palette (Cmd+K)]  [Bell] [Av]|
+---------------+--------------------------------------------------------------+
| SIDEBAR       |  PAGE HEADER (title, breadcrumb, primary action button)     |
|               |----------------------------------------------------------  |
| > Dashboard   |                                                            |
|   Projects    |                                                            |
|   Campaigns   |               MAIN WORKSPACE CONTENT                      |
|   Templates   |          (page-specific, see Section 5)                    |
|   Reports     |                                                            |
|   Marketplace |                                                            |
|   Integrations|                                                            |
| ------------- |                                                            |
|   Settings    |                                                            |
|   Team        |                                                            |
+---------------+--------------------------------------------------------------+
```

**Top Navigation**: fixed, 56px height. Contains the logo, a Workspace Switcher (a Combobox listing every organization the user belongs to, matching the API's multi-organization membership model), the Command Palette trigger, a notification bell with an unread-count dot, and the profile menu avatar. It never scrolls out of view, since switching workspace context needs to be reachable from any depth in the application.

**Sidebar**: 240px expanded, collapsible to a 64px icon-only rail (persisted per user in local preference, not a session-only toggle). Primary navigation items map directly to the Pages in Section 5. A visual divider separates day-to-day navigation (Dashboard through Integrations) from account-level navigation (Settings, Team), so the two categories are never visually confused.

**Workspace (main content area)**: the page-specific region described per page in Section 5. Maintains a consistent 24px outer padding and a page header pattern (title, optional breadcrumb, optional primary action button, right-aligned) across every page for predictability.

**Execution Panel**: not a persistent layout region, a page-level pattern used specifically by the Workflow Execution page (Section 5), described fully in Section 9.

```
EXECUTION PANEL (used inside the Workflow Execution page)
+------------------------------------------------+------------------------+
|  WORKFLOW TIMELINE (vertical, left rail)         |  DETAIL PANE           |
|  o Research          done      2m 14s            |  (contents depend on   |
|  o Strategy          done      1m 02s            |   which timeline step  |
|  > Copywriter        running   0:38               |   is selected; shows   |
|  o Carousel          pending                      |   agent output, logs,  |
|  o Validation         pending                      |   or tool calls, see   |
|  o Awaiting Approval   pending                       |   Section 9)          |
+------------------------------------------------+------------------------+
```

**Workflow Timeline**: the vertical rail shown above, always visible on the left of the Execution Panel while a run is active, collapsible to a horizontal compact strip on narrower viewports (Section 13).

**Notifications**: a slide-over panel triggered from the bell icon, listing recent notifications with the same visual weight as the API's `Notifications` module, never a full-page destination in Version 1, since the volume does not yet justify one.

**Profile Menu**: a dropdown from the top-right avatar containing profile settings, theme toggle, and sign out, kept deliberately small since account-level actions live on the Settings page, not duplicated in a menu.

**Settings**: reached via the sidebar, not the profile menu, to avoid two competing entry points to the same destination.

**Command Palette**: `Cmd+K` / `Ctrl+K` global trigger, opening a centered `elevation-2` modal. Supports three modes: navigation ("go to campaigns"), action ("new campaign", "invite teammate"), and search (campaigns, brands, past content by keyword), switching mode based on a `>` prefix convention for actions, matching the pattern established by Linear and Vercel's own dashboards, which this document explicitly takes as a visual reference point.

---

## 5. Pages

Every page below lists the APIs it depends on using the exact resource names from the API design document, so a frontend engineer can trace every screen back to a concrete backend contract with no ambiguity.

### Landing

**Purpose**: marketing entry point, unauthenticated. **Components**: hero, product screenshot/demo embed, feature grid, pricing teaser, footer. **User actions**: sign up, log in, watch demo. **Navigation**: to Signup, Login. **Required APIs**: none, fully static.

### Login

**Purpose**: authenticate an existing user. **Components**: email/password form (React Hook Form + Zod schema), OAuth buttons (Google, Microsoft), "forgot password" link. **User actions**: submit credentials, initiate OAuth. **Navigation**: to Dashboard on success, to Signup. **Required APIs**: `POST /api/v1/auth` (session exchange after the OAuth/JWT provider redirect), `GET /api/v1/auth/me`.

### Signup

**Purpose**: create a new account and first organization. **Components**: account form, organization name field, role defaults to Owner. **User actions**: submit, verify email. **Navigation**: to Dashboard (onboarding checklist state) on success. **Required APIs**: `POST /api/v1/organizations`, `GET /api/v1/auth/me`.

### Dashboard

**Purpose**: the default landing page after login, answers "what needs my attention." **Components**: full breakdown in Section 8. **User actions**: jump into any active workflow, start a new campaign, review notifications. **Navigation**: hub page, links to Projects, Campaigns, Reports. **Required APIs**: `GET /api/v1/campaigns?status=awaiting_approval`, `GET /api/v1/workflow-executions?status=running`, `GET /api/v1/notifications`.

### Projects

**Purpose**: the Brand Profiles list, framed to the user as "Projects" or "Clients" depending on workspace type (a naming decision left to a workspace-level setting, since an in-house marketing team and an agency use different vocabulary for the same underlying entity). **Components**: card grid, search, "new project" primary action. **User actions**: create, open, archive a brand profile. **Navigation**: to Campaign list scoped to a project. **Required APIs**: `GET /api/v1/brands`, `POST /api/v1/brands`.

### Workflow Builder

**Purpose**: the Command Box experience where a goal becomes a campaign. Full UX flow in Section 7. **Components**: Command Box, brand selector, constraints panel (post count, deadline), plan preview. **User actions**: type goal, select brand, review the proposed plan, confirm. **Navigation**: to Workflow Execution on confirm. **Required APIs**: `POST /api/v1/campaigns`, `GET /api/v1/campaigns/{id}/goal`.

### Workflow Execution

**Purpose**: the Execution View, described fully in Section 9. **Components**: Workflow Timeline, Detail Pane, Approval Modal trigger. **User actions**: pause, resume, cancel, retry, approve, request changes. **Navigation**: to Campaign Detail once completed. **Required APIs**: `GET /api/v1/workflow-executions/{id}`, `WS /api/v1/workflow-executions/{id}/stream`, `GET /api/v1/workflow-executions/{id}/logs`, `POST .../pause`, `POST .../resume`, `POST .../retry`, `POST /api/v1/campaigns/{id}/approve`.

### Campaign Detail

**Purpose**: the finished (or in-review) artifact view, content-first rather than process-first, the natural landing page once a campaign is no longer actively executing. **Components**: post cards with draft/approved states, carousel slide viewer, image prompt list with copy buttons, publishing schedule. **User actions**: edit content inline, approve, view analytics. **Navigation**: to Reports (analytics), back to Projects. **Required APIs**: `GET /api/v1/campaigns/{id}/content`, `PATCH /api/v1/generated-content/{id}`, `GET /api/v1/campaigns/{id}/publishing-jobs`.

### Reports

**Purpose**: analytics rollups. **Components**: metric cards, engagement charts (Recharts), campaign comparison table. **User actions**: filter by date range and brand, export. **Navigation**: from Dashboard, Campaign Detail. **Required APIs**: `GET /api/v1/campaigns/{id}/analytics`, `GET /api/v1/brands/{id}/analytics/summary`.

### Templates

**Purpose**: surfaces the Procedural Memory concept from the platform's architecture as reusable starting points, workflows that have succeeded repeatedly, promoted into named templates a user can launch from directly rather than typing a fresh goal every time. **Components**: template card grid, preview. **User actions**: launch from template, save current campaign as template. **Navigation**: to Workflow Builder, pre-filled. **Required APIs**: `GET /api/v1/brands/{id}/memory` (filtered to template-type entries).

### Marketplace

**Purpose**: Version 2+ surface for community or partner-built templates and tool connections, stubbed with an empty state and waitlist form in Version 1, matching the Vision document's phased roadmap rather than being fully built ahead of need.

### Integrations

**Purpose**: manage Tool Connections. **Components**: connection cards per platform (LinkedIn in Version 1) with status, connect/disconnect actions. **User actions**: authorize, revoke. **Navigation**: from Settings. **Required APIs**: `GET /api/v1/brands/{id}/tool-connections`, `POST .../linkedin/authorize`, `DELETE .../linkedin`.

### Billing

**Purpose**: plan, usage, and payment management. **Components**: plan card, usage meters (mapped to the rate limiting tiers in the API design), invoice history. **User actions**: upgrade plan, update payment method. **Navigation**: from Settings. **Required APIs**: `GET /api/v1/organizations/{id}` (plan/billing status).

### Settings

**Purpose**: organization-level configuration. **Components**: tabbed layout (General, Members, Brand Defaults, Danger Zone). **User actions**: rename organization, manage members, delete organization. **Navigation**: hub for Team, Integrations, Billing. **Required APIs**: `PATCH /api/v1/organizations/{id}`, `GET .../members`.

### Profile

**Purpose**: individual user preferences. **Components**: name, avatar, notification preferences, theme toggle. **Required APIs**: `GET /api/v1/users/me`, `PATCH /api/v1/users/me`.

### Team

**Purpose**: member management, a focused sub-view of Settings surfaced directly in the sidebar given how frequently agencies invite contractors. **Components**: member table with role Select per row, invite form. **User actions**: invite, change role, remove. **Required APIs**: `GET /api/v1/organizations/{id}/members`, `POST/PATCH/DELETE` on the same resource.

### Notifications

**Purpose**: full notification history, reached from "view all" inside the Notification slide-over. **Components**: filterable list. **Required APIs**: `GET /api/v1/notifications`, `PATCH /api/v1/notifications/{id}`.

### Admin

**Purpose**: platform-internal only, gated by the `is_platform_admin` claim described in the API design, not visible to any customer-facing role. **Components**: prompt template management, feature flag control, cross-organization search for support purposes. **Required APIs**: `GET/POST /api/v1/admin/prompt-templates`.

### 404

**Purpose**: not-found fallback. **Components**: a message, a link back to Dashboard, matching the shell's visual language rather than a fully custom illustration-heavy page, since a workflow tool's 404 page is a rare event, not a branding opportunity worth over-investing in.

---

## 6. Components

Grouped by how specific they are to WorkflowOS, since the primitives (Button, Input, Card) are close to shadcn/ui defaults and do not need much elaboration, while the domain components are where the actual design work lives.

### Primitives (shadcn/ui-based, minimal customization)

| Component | Notes |
|---|---|
| Button | Variants per Section 3. Loading state replaces label with a spinner at fixed width, so the button never changes size on click. |
| Input | Standard text input, always paired with a `Label` and, where relevant, an inline Zod validation error below it. |
| Card | Base container per Section 3. |
| Search | A single reusable component wrapping an Input with a debounced `onSearch` callback (300ms), used identically on Projects, Campaign lists, and the Command Palette's search mode. |
| Filters | A `Popover` containing Select and date-range controls, applied as URL query parameters (Section 11) so filtered views are shareable links, not local-only state. |
| Navigation | The Sidebar and Top Navigation from Section 4, implemented as layout components, not page components. |
| Tables | Built on `@tanstack/react-table` for sorting and pagination logic, styled with the Table primitives from Section 3. |
| Pagination | Cursor-based, matching the API's pagination model exactly, "Load more" rather than numbered pages, since cursor pagination has no concept of a page number. |
| Dialogs | shadcn/ui `Dialog`, `elevation-2`, used for anything that requires the user's full attention before proceeding, including the Approval Modal below. |
| Toasts | `elevation-3`, auto-dismiss after 5 seconds except for error toasts, which persist until manually dismissed, since an error a user cannot act on immediately should not disappear before they notice it. |

### Domain components

**Workflow Card**: used on the Dashboard and Projects pages. Shows campaign name, brand, current status Badge, a compact progress indicator (a segmented bar, one segment per workflow step, filled proportionally to completion), and a relative timestamp. Clicking anywhere on the card navigates to Workflow Execution if running, or Campaign Detail if completed, so the destination is state-dependent, not fixed.

**Execution Timeline**: the vertical rail in Section 4's Execution Panel wireframe. Each row is a Workflow Step with an icon (per-agent icon, matching the agent identity throughout the Execution View for visual consistency), a label, a status Badge, and elapsed or total duration. The currently active row uses the `Card` "Highlighted" variant and a subtle pulsing dot rather than a spinner, to avoid visual noise across up to eight rows.

**Agent Status Card**: appears in the Detail Pane when a timeline step is selected. Shows the agent's name, a one-line description of what it does (sourced directly from the AI Agent API contracts, e.g. "Reviews generated content against brand voice and negative constraints"), its input summary, and its output once complete.

**Progress Bar**: two variants, determinate (used once step count is known, the common case) and indeterminate (used only during the brief interval before the Planner returns a step count, since Version 1 has a fixed DAG and step count is knowable almost immediately).

**Logs Viewer**: a monospace, dark-surfaced panel (using `text-mono` regardless of active theme, since logs read poorly in a light monospace block) with one line per event, timestamped, filterable by step. Reads directly from `GET /workflow-executions/{id}/logs`, using the human-readable messages the API design specifies, never raw exception text.

**Approval Modal**: triggered from the Execution View when a campaign reaches `awaiting_approval`, and also reachable from a Dashboard notification. Shows every piece of Generated Content requiring approval, each individually checkable, with "Approve Selected," "Approve All," and "Request Changes" (which opens a text field for reviewer notes, mapped to `POST /campaigns/{id}/request-changes`). This modal is intentionally not dismissible by clicking outside it, given its role as the platform's single mandatory human checkpoint.

**Command Box**: the large textarea on the Workflow Builder page. Auto-expands up to 8 lines, includes a character counter matching the API's 2000-character goal limit, and a brand Combobox inline above it. Submitting triggers a brief inline "Understanding your goal..." state before transitioning to the plan preview (Section 7).

**Prompt Editor**: used inside the Admin page for prompt template management. A code-editor-style textarea (monospace, line numbers) with a version history dropdown and a diff view between versions, since prompt changes are exactly the kind of edit that benefits from a visible diff before promoting to active.

**Campaign Card**: a denser variant of Workflow Card used specifically in Campaign Detail's related-campaigns rail, dropping the progress indicator since the campaign is already complete by the time this variant appears.

**Metric Cards**: used on Dashboard and Reports. A label, a large numeric value, and an optional trend indicator (a small colored arrow plus percentage change), never a sparkline inline in the card itself, sparklines live in the full chart views to avoid two conflicting scales on one screen.

**Charts**: Recharts-based, three chart types cover every need in Version 1: a line chart (engagement over time), a bar chart (campaign comparison), a donut chart (content status breakdown). All charts share one color mapping, drawn from the same status colors used in Badges, so a red segment always means the same thing whether the user is looking at a Badge or a chart.

---

## 7. Workflow Builder UX

```
STEP 1: GOAL              STEP 2: AI PLANS            STEP 3: PREVIEW & APPROVE
+-----------------+       +-----------------+          +------------------------+
| Brand: [Kube v]  |       |  Understanding    |          | Proposed plan:          |
| Goal:            | ----> |  your goal...      |  ----->  | - Research (2m est)      |
| "Launch a        |       |  (inline, ~2-4s)    |          | - Strategy (1m est)       |
|  LinkedIn         |       |                     |          | - 5 posts, 1 carousel      |
|  campaign..."     |       |                     |          | - Validation, then         |
| [Constraints v]   |       |                     |          |   your approval           |
| [Start >]         |       |                     |          | [Edit] [Start Execution]  |
+-----------------+       +-----------------+          +------------------------+
                                                                     |
                                                                     v
                                                    STEP 4: EXECUTION (Section 9)
```

**User enters goal**: the Command Box accepts free text, with the brand and constraints (post count, deadline, carousel inclusion) set in a lightweight panel above it rather than buried in an "advanced options" disclosure, since these three fields are set on nearly every campaign and hiding them would cost more clicks than it saves.

**AI plans**: submitting calls `POST /api/v1/campaigns`, which returns in under a second per the API design's `202 Accepted` contract. The UI shows a brief, honest "Understanding your goal" state rather than a long fake-progress animation, since the actual planning work happens asynchronously and the UI should not pretend otherwise.

**Execution preview**: once the Campaign Goal (the parsed, structured intent) is available, the UI shows the proposed plan as a simple ordered list, not yet the full timeline, before any agent has run. This is a deliberate pause point, giving the user a chance to catch a misunderstood goal (wrong brand selected, wrong post count) before spending any execution time or LLM cost on it. It is a soft checkpoint, editable and skippable, distinct from the hard Approval Modal checkpoint later in the flow.

**Approval**: clicking "Start Execution" is a lightweight confirmation, not the same interaction as the Approval Modal in Section 6. Its label is deliberately different ("Start Execution," never "Approve") so a user never confuses "I am about to let this run" with "I am about to let this publish."

**Execution starts, timeline updates, logs appear, results generated**: control passes to the Workflow Execution page, described fully in Section 9. The transition from Workflow Builder to Workflow Execution is a route change, not a modal, so the URL is shareable and the browser back button behaves predictably, both of which matter for an account manager who might switch between three concurrent campaigns across browser tabs.

---

## 8. Dashboard Design

The Dashboard answers one question: what needs my attention right now. Widgets are ordered top to bottom by how directly they answer that question, not by category.

```
+-----------------------------------------------------------------------+
|  Needs Your Approval (3)                          [View all >]         |
|  [Campaign Card] [Campaign Card] [Campaign Card]                        |
+-----------------------------------------------------------------------+
|  Active Workflows (2)                             [View all >]          |
|  [Workflow Card - Copywriter running]  [Workflow Card - Research]        |
+---------------------------+---------------------------------------------+
|  AI Usage this month       |  Upcoming Schedules                          |
|  [donut: credits used]      |  Jul 12 09:00  Kubernetes post 1              |
|  340 / 500 credits           |  Jul 13 09:00  Kubernetes post 2               |
+---------------------------+---------------------------------------------+
|  Recent Campaigns                                  [View all >]           |
|  [Campaign Card] [Campaign Card] [Campaign Card] [Campaign Card]           |
+-----------------------------------------------------------------------+
|  Quick Actions: [+ New Campaign]  [Invite teammate]  [Browse templates]     |
+-----------------------------------------------------------------------+
```

**Needs Your Approval**: the highest-priority widget, always first, using Workflow Cards filtered to `awaiting_approval`. Empty state reads "Nothing waiting on you," a deliberately calm, positive empty state rather than a blank space, since this is the widget a returning user checks first.

**Active Workflows**: currently running campaigns, each card showing live progress via the same WebSocket connection the Execution View itself uses, so a user can see a step complete on the Dashboard without opening the campaign.

**AI Usage / Credits**: a compact usage meter against the organization's plan limit (mapped to the rate limiting tiers in the API design), with a warning state (amber, then red) as the organization approaches its limit, surfaced here rather than only on the Billing page, since running out of credit mid-campaign is a failure mode worth warning about before it happens.

**Upcoming Schedules**: the next several `Publishing Jobs` across all campaigns, giving an at-a-glance publishing calendar without navigating to each Campaign Detail page individually.

**Recent Campaigns**: a simple recency-ordered list, the fallback content once the higher-priority widgets are empty or fully reviewed.

**Quick Actions**: persistent row, always visible regardless of what else is on the Dashboard, since "start a new campaign" is the single most common action a returning user takes.

---

## 9. Execution View

This is the page the entire product philosophy in Section 1 is built to justify. If this screen fails to make autonomous execution feel legible and trustworthy, nothing else in this document matters.

```
+----------------------------------------------------------------------------+
| Kubernetes Security Campaign                    [Pause] [Cancel]  Status:   |
| Goal: "Launch a LinkedIn campaign about..."                     GENERATING  |
+---------------------+--------------------------------------------------------+
| TIMELINE              |  DETAIL PANE: Copywriter Agent                         |
|                        |  --------------------------------------------------  |
| o Research    2m 14s   |  Status: running (0:38 elapsed)                        |
|   done                 |  Input: Campaign Architecture (5 posts, 1 carousel)     |
|                        |  Output so far: 3 of 5 posts drafted                    |
| o Strategy    1m 02s    |                                                          |
|   done                  |  [Post 1 - draft]                                        |
|                          |  "Kubernetes RBAC misconfigurations are the..."           |
| > Copywriter   0:38       |  [Post 2 - draft] [Post 3 - draft]                         |
|   running                  |                                                             |
|                              |  --- Tool Calls (2) -----------------------------------  |
| o Carousel     pending        |  qdrant.search  brand_voice_examples     ok  120ms       |
|                                 |  llm.generate    copywriter_v3          ok  1.4s         |
| o Validation    pending          |                                                           |
|                                    |  --- Validation ---------------------------------------  |
| o Awaiting Approval pending          |  Not yet run                                              |
+---------------------+------------------------------------------------------------+
|  LOGS (collapsible, full width)                                                     |
|  10:02:03  Campaign created, workflow enqueued                                        |
|  10:02:05  Research Agent started                                                       |
|  10:04:19  Research Agent completed, 12 sources found                                     |
|  10:04:20  Strategy Agent started                                                          |
+----------------------------------------------------------------------------+
```

Each region maps to a specific data need and a specific principle from Section 1 or 2:

**Goal**: shown persistently at the top of the page, never scrolled out of view, because every other panel on this page is an answer to the question this goal asks. A user should never have to scroll up to remember what they are watching happen.

**Planner**: represented implicitly by the plan the Timeline is built from, rather than its own panel. The Planner's output (the Campaign Architecture) is visible the moment a user clicks into the Strategy step in the Timeline, which is where a curious user would naturally look for it.

**Agents**: each Timeline row corresponds one-to-one with an agent from the API design's AI Agent module (Research, Strategy, Copywriter, Carousel, Image Prompt, Validator, and, later, Publisher and Analytics once the campaign moves past approval). Icons are consistent across the Timeline, the Agent Status Card, and the Logs, so a user builds a fast visual vocabulary for "which agent is talking."

**Execution Steps**: the Timeline itself, described in Section 6.

**Current Status**: the top-right Badge, using the exact status enum from the Workflow Executions API (`pending`, `researching`, `strategizing`, `generating_content`, `awaiting_approval`, `publishing`, `completed`, `failed`, `cancelled`), so there is never a translation layer between what the backend calls a state and what the UI displays, reducing the chance of the two drifting out of sync.

**Logs**: the full-width panel at the bottom, collapsible to reclaim vertical space once a user has confirmed the run is healthy, but never fully hidden, matching the Trust Through Transparency principle.

**Tool Calls**: nested inside the Detail Pane for the currently selected step, each call shown with its name, result (ok or error), and latency, directly satisfying the audit and explainability requirements from the API design's security section, surfaced to the actual user rather than only existing in a Langfuse trace a developer would need to go find.

**Validation**: its own subsection within the Detail Pane once the Validator Agent has run, showing pass or fail per piece of content with the specific flagged span and rule violated (Section 6 of the API design's contract for the Validator agent), never a bare "failed."

**Results**: once a run reaches `awaiting_approval` or later, the Detail Pane's default selected step becomes the results summary rather than the last agent to run, since a completed run's most useful default view is what it produced, not the mechanics of how it got there.

**History**: a tab alongside the live Timeline view, listing past runs for the same campaign (relevant after a retry), each collapsed by default with the same Timeline visualization available on expand, so debugging a retried campaign does not require leaving the page.

---

## 10. State Management

Zustand, six focused stores rather than one large store, each with a narrow, testable responsibility. No store persists sensitive data (tokens) to `localStorage`, see Section 16.

| Store | Holds | Notes |
|---|---|---|
| **Global Store** | Current organization ID, sidebar collapsed state, command palette open state | The only store touched by more than one feature area; kept deliberately small |
| **User Store** | Current user profile, role within the active organization, notification preferences | Populated from `GET /auth/me` on load, refetched on organization switch |
| **Workflow Store** | The list of campaigns/workflows for list views (Dashboard, Projects), pagination cursors | Server state that changes on user action (create, cancel); not the live execution state, which has its own store below |
| **Execution Store** | The single actively-viewed workflow execution's live state: current step, timeline, logs buffer | Populated by the initial `GET /workflow-executions/{id}` and then patched incrementally by WebSocket messages, never fully refetched on every update, to keep the Execution View responsive under a high message rate |
| **Notification Store** | Unread count, recent notifications for the slide-over panel | Updated by both polling and, where the backend supports it, push |
| **Theme Store** | Dark/light preference | Persisted to `localStorage` (non-sensitive), read before first paint via a blocking script to avoid a flash of the wrong theme |

Server state that does not need real-time updates (Brands, Reports, Settings) is deliberately kept out of Zustand entirely and handled by React Query instead, matching the split described in the API design's REST layer: Zustand owns client/UI state, React Query owns server state and its caching, and the Execution Store is the one intentional exception, because its update pattern (WebSocket push) does not fit React Query's fetch-based cache model cleanly.

---

## 11. Routing Structure

```
app/
├── (marketing)/
│   └── page.tsx                       Landing
├── (auth)/
│   ├── login/page.tsx
│   └── signup/page.tsx
├── (app)/                              Authenticated shell layout wraps everything below
│   ├── layout.tsx                       Top Nav + Sidebar (Section 4)
│   ├── dashboard/page.tsx
│   ├── projects/
│   │   ├── page.tsx                       Projects list
│   │   └── [brandId]/page.tsx               Single brand, its campaigns
│   ├── campaigns/
│   │   ├── new/page.tsx                       Workflow Builder
│   │   └── [campaignId]/
│   │       ├── execution/page.tsx               Workflow Execution
│   │       └── page.tsx                          Campaign Detail
│   ├── templates/page.tsx
│   ├── marketplace/page.tsx
│   ├── reports/page.tsx
│   ├── integrations/page.tsx
│   ├── billing/page.tsx
│   ├── settings/
│   │   ├── page.tsx                                General
│   │   ├── team/page.tsx
│   │   └── danger-zone/page.tsx
│   ├── profile/page.tsx
│   ├── notifications/page.tsx
│   └── admin/page.tsx                                 Gated by is_platform_admin
├── not-found.tsx                                        404
└── api/                                                   Next.js route handlers, thin
                                                             proxies to the FastAPI backend
                                                             where a server-side secret must
                                                             stay off the client (OAuth
                                                             callback exchange), not a
                                                             general-purpose API layer
```

Route groups (`(marketing)`, `(auth)`, `(app)`) separate layouts without affecting the URL, matching the App Router convention directly. The `(app)` group's `layout.tsx` is where the authenticated shell from Section 4 lives, meaning every page inside it inherits Top Navigation and Sidebar automatically, and no individual page component ever renders its own navigation chrome.

---

## 12. Folder Structure

```
src/
├── app/                     Next.js App Router routes only (Section 11). Route files stay
│                             thin: data fetching via hooks, layout via components, nothing
│                             else lives here.
├── components/
│   ├── ui/                    shadcn/ui primitives, largely unmodified from the generator
│   ├── domain/                 WorkflowCard, ExecutionTimeline, ApprovalModal, and every
│   │                            other component from Section 6 that is specific to
│   │                            WorkflowOS rather than a generic UI primitive
│   └── layout/                   TopNav, Sidebar, CommandPalette
├── hooks/
│   ├── queries/                     React Query hooks, one file per API resource
│   │                                  (useCampaigns, useWorkflowExecution), each wrapping
│   │                                  the corresponding services/ function
│   └── use-workflow-stream.ts          The WebSocket subscription hook powering the
│                                         Execution Store
├── services/                 Thin API client functions, one per backend resource, the only
│                               place a fetch call to /api/v1/* is allowed to exist; hooks
│                               and components never call fetch directly
├── stores/                   The six Zustand stores from Section 10, one file each
├── types/                    TypeScript types generated from the OpenAPI schema (per the
│                               API design's SDK generation strategy), plus any purely
│                               frontend-only types (component prop types live beside their
│                               components instead)
├── lib/                      Cross-cutting utilities with no UI or API awareness: date
│                               formatting, the Zod schema helpers, the color-token-to-Tailwind
│                               mapping used by Badge and status colors
├── styles/                   Tailwind config, the semantic color token definitions from
│                               Section 3, global CSS
├── providers/                 React Query provider, Theme provider, WebSocket provider,
│                                composed once in the root layout
└── middleware.ts               Next.js middleware: route protection (Section 16),
                                  redirecting unauthenticated users away from the (app)
                                  route group before a page component ever renders
```

The `services/` and `hooks/queries/` split exists specifically so the API contract (services) and the caching/loading-state behavior (hooks) can change independently: a backend endpoint rename touches one `services/` file, a caching strategy change touches one `hooks/queries/` file, and components touch neither.

---

## 13. Responsive Design

WorkflowOS is designed desktop-first, not because mobile does not matter, but because the primary persona (an account manager managing several concurrent campaigns) does this work at a desk. Mobile and tablet get a genuinely usable, monitoring-oriented experience, not a stripped-down afterthought, but the Workflow Builder's constraint panel and the Execution View's side-by-side Timeline and Detail Pane are desktop-optimized layouts that adapt rather than shrink.

| Breakpoint | Width | Layout change |
|---|---|---|
| `mobile` | < 640px | Sidebar becomes a bottom-sheet drawer triggered by a hamburger icon; Execution View's Timeline and Detail Pane stack vertically, Timeline collapses to a horizontal, swipeable step strip above the Detail Pane rather than a vertical rail |
| `tablet` | 640 to 1024px | Sidebar collapses to the icon-only rail by default (user can still expand it); Dashboard widgets go from a 2-column to a single-column stack |
| `laptop` | 1024 to 1440px | Full desktop layout, Execution View's two-column Timeline/Detail split remains but at reduced padding |
| `desktop` | > 1440px | Execution View expands to full viewport width per Section 3's grid exception; all other pages stay capped at 1280px content width, centered |

**Navigation changes**: below `tablet`, the Top Navigation's Workspace Switcher collapses to an icon that opens a full-screen switcher rather than an inline dropdown, since a dropdown anchored to a small touch target performs poorly on mobile viewports.

**Cards**: Workflow Cards and Campaign Cards go from a 3-column grid (desktop) to 2-column (tablet) to a single column (mobile), with no change to the card's internal layout, only the grid wrapping it, keeping card component code identical across breakpoints.

**Tables**: below `tablet`, dense tables (Section 3) switch to a card-per-row layout instead of horizontal scroll, since horizontal scroll on a data table is a common but genuinely poor mobile pattern; each row's columns become labeled key-value pairs stacked vertically within a card.

---

## 14. Accessibility

- **Keyboard navigation**: every interactive element reachable via `Tab`, in a logical order matching visual layout; the Command Palette (`Cmd+K`) and its internal arrow-key navigation are the primary keyboard-first surface, but standard tab order is never sacrificed elsewhere in favor of custom shortcuts.
- **ARIA**: the Execution Timeline uses `role="list"` with each step as `role="listitem"`, and the currently active step carries `aria-current="step"`, so a screen reader user gets an explicit, unambiguous signal of progress that sighted users get visually through the Highlighted card variant. The Approval Modal uses `role="alertdialog"` given its mandatory, blocking nature, not the plain `dialog` role used elsewhere.
- **Focus management**: opening any Dialog (including the Approval Modal) moves focus to the dialog's first focusable element and traps it there until closed; closing returns focus to the element that triggered it, never to the document body.
- **Contrast**: enforced at the design token level per Section 3, so no component-level exception can silently ship a failing contrast ratio.
- **Screen readers**: live-updating regions, the Timeline's status changes and the Logs Viewer's new entries, use `aria-live="polite"`, never `"assertive"`, since a constant stream of assertive announcements during an active execution would make the page unusable with a screen reader; the one deliberate exception is a `failed` status transition, which uses `aria-live="assertive"` because a run failing is exactly the kind of update that should interrupt.

---

## 15. Performance

- **Server components by default**: every page component in `app/` is a Server Component unless it needs interactivity (state, event handlers, the WebSocket connection), in which case only the smallest necessary subtree is marked `"use client"`. The Execution View's static header (goal, campaign name) is a Server Component; the Timeline and Detail Pane, which need live state, are Client Components.
- **Streaming**: pages with genuinely slow initial data (Reports, with its analytics aggregation) use `loading.tsx` and React Suspense boundaries around the chart components specifically, so the page shell and navigation render immediately while only the chart area shows a skeleton.
- **Dynamic imports / code splitting**: the Prompt Editor (Admin page), Recharts-based chart components, and Framer Motion's more elaborate animation variants are all dynamically imported, since none of the three are needed on first paint for the majority of pages and Recharts in particular is a meaningfully large dependency to include in the main bundle.
- **Lazy loading**: below-the-fold Dashboard widgets (Recent Campaigns, given it is the lowest-priority widget per Section 8) lazy-load on scroll into view rather than blocking the initial Dashboard render behind the higher-priority Needs Your Approval widget.
- **Image optimization**: `next/image` throughout, with brand logos and any user-uploaded media assets served through Next.js's image optimization pipeline rather than the raw Supabase Storage URL, so resizing and format negotiation (WebP/AVIF) happen automatically.
- **Caching**: React Query's stale-while-revalidate model for all server state, with query keys namespaced by organization ID so switching workspaces never serves stale cross-tenant data from cache, a direct frontend mirror of the backend's tenant isolation requirement.

---

## 16. Security

- **Authentication**: JWT-based per the stated stack, obtained through the OAuth flow and exchanged via the Next.js route handler in `app/api/`, never handled with a client-side OAuth library that would expose a client secret.
- **Token storage**: the session token is stored in an `httpOnly`, `Secure`, `SameSite=Lax` cookie, set by the Next.js route handler, never in `localStorage` or a client-readable cookie. This is a deliberate, non-negotiable choice given the platform's own architecture: an XSS vulnerability anywhere in a large, evolving frontend codebase should not be able to exfiltrate a session token, and `httpOnly` cookies are immune to that specific attack class in a way `localStorage` is not.
- **Protected routes**: enforced twice, once in `middleware.ts` at the edge, before any page component renders, and once inside each Server Component's data fetch, which will itself fail with a `401` if the session is invalid, so a client-side bypass of the middleware check alone cannot render authenticated content.
- **CSRF**: the `SameSite=Lax` cookie setting is the primary mitigation, appropriate because the API is JSON-only and does not accept the traditional form-encoded submissions CSRF attacks rely on; state-changing requests additionally require a custom header the browser will not attach cross-origin without an explicit CORS allowance, which the API design's CORS policy does not grant.
- **XSS**: React's default JSX escaping covers the overwhelming majority of surfaces; the two places raw HTML rendering is ever considered, the Logs Viewer and any future rich-text content preview, use `DOMPurify` explicitly rather than `dangerouslySetInnerHTML` on unsanitized input, and this is treated as a hard rule, not a case-by-case judgment call.
- **Permissions and role-based UI**: the User Store's role field (Owner, Editor, Viewer, per the API design's RBAC model) gates UI at the component level through a single `<RequireRole role="editor">` wrapper used consistently everywhere a role check is needed, rather than scattered conditional rendering, so an auditor can grep the codebase for every role-gated surface in one search. Hiding a button from a Viewer is a UX convenience, not a security boundary; every gated action is independently enforced server-side per the API design's authorization model, and the frontend never assumes otherwise.

---

## 17. Error Handling

- **Empty states**: every list view (Campaigns, Projects, Notifications, Templates) has a purpose-written empty state, an icon, a one-line explanation, and, where relevant, a primary action ("Create your first campaign"), never a bare "No results" string.
- **Loading states**: skeleton screens matching the eventual content's layout, not a generic spinner, used on every page's initial load; the Execution View is the one exception, where an indeterminate Progress Bar is more honest than a skeleton, since its content genuinely does not exist yet in any form until the Planner returns.
- **Error states**: a distinct visual pattern from empty states (a warning-colored icon, not a neutral one), always including the `request_id` from the API's error envelope in small `text-caption` text, so a user reporting an issue can hand support a specific, traceable identifier rather than a screenshot alone.
- **Retry states**: any failed data fetch shows an inline "Something went wrong, retry" action scoped to just that panel, never a full-page reload prompt, since a failure in, for example, the Reports chart should not take down the rest of the Dashboard.
- **Offline mode**: a persistent top banner appears when the browser's `navigator.onLine` goes false or a fetch fails with a network error specifically (distinct from a `5xx` response), warning that live updates (the WebSocket-driven Execution View in particular) are paused; the WebSocket client attempts reconnection with exponential backoff and the banner clears automatically once reconnected.

---

## 18. Animation Guidelines

Framer Motion, used deliberately and sparingly, since an execution-monitoring tool that overuses motion undermines its own credibility.

- **Page transitions**: a brief 150ms fade, no slide or scale, kept intentionally subtle so navigation feels instant rather than like a presentation.
- **Cards**: a small scale (1.0 to 1.02) and elevation increase on hover for Interactive cards, 100ms ease-out, no animation on non-interactive cards.
- **Loading**: skeleton screens use a slow (1.5s) shimmer, not a bouncing or pulsing effect, matching the calm, low-noise visual language from Section 2.
- **Execution Timeline**: when a step transitions from `pending` to `running`, the status Badge crossfades (200ms) rather than instantly swapping, and the connecting line between timeline nodes fills with a brief (400ms) draw-on animation, the one place in the entire application where a slightly more expressive animation is justified, since it directly visualizes the product's core value proposition: progress happening.
- **Agent Movement**: no literal animated agent avatars moving across the screen. This is worth stating explicitly as a guideline, not just an omission, because it is a common pattern in AI-product marketing sites and would read as gimmicky inside a serious execution-monitoring tool; the pulsing dot on the active Timeline row (Section 6) is the entire visual vocabulary for "an agent is working."
- **Micro interactions**: button press states use a 1px scale-down (50ms), toast entrances slide up 8px while fading in (200ms), and that is close to the complete list; this document treats animation restraint as a design principle in its own right, not an oversight to fill in later.

---

## 19. Future UI Vision

**Version 1**: the fixed-DAG execution model described throughout this document. One vertical (LinkedIn marketing campaigns), one Timeline shape, a known, finite set of agents. The Execution View's Timeline is a straight vertical list because the underlying workflow is a straight DAG.

**Version 2**: as the backend's Vision document roadmap introduces multi-channel campaigns and closed-loop analytics feeding back into future plans, the Timeline gains branches, still readable as a graph, not yet a fully dynamic one, and the Reports page gains a "what changed since last time" view surfacing the Procedural Memory system's learned adjustments directly, rather than requiring a user to infer them from campaign performance alone.

**Version 3**: once the backend moves toward dynamic, MCP-discovered tools and workflow graphs generated per-goal rather than fixed at build time (per the Technical Architecture document's stated evolution path), the Execution View's Timeline can no longer be a hardcoded set of known agent icons. It becomes a generic graph renderer capable of displaying an arbitrary plan shape, with agent identity resolved dynamically from the Tool Registry rather than from a fixed icon map, the single largest frontend architecture change this document anticipates, and the reason the Timeline component (Section 6) should be built now with a data-driven step model rather than hardcoded step types, even though Version 1 only ever needs eight fixed steps.

---

## 20. Appendix

**Naming conventions**: components in `PascalCase` matching their filename (`WorkflowCard.tsx`); hooks prefixed `use` (`useWorkflowExecution`); Zustand stores suffixed `Store` (`executionStore.ts`); Zod schemas suffixed `Schema` (`campaignGoalSchema`); API service functions named as verbs matching the HTTP action (`createCampaign`, `approveCampaign`, never `campaignPost`).

**Component conventions**: every domain component (Section 6) accepts its data via typed props derived from the generated API types (Section 12), never fetches its own data internally, keeping components pure and independently testable; data fetching lives exclusively in `hooks/queries/`.

**Coding standards**: TypeScript strict mode, no `any` outside of explicitly justified, commented exceptions; ESLint with the Next.js and `react-hooks` rule sets as a merge-blocking check, not a warning.

**Tailwind conventions**: utility classes only, no custom CSS files except `styles/globals.css` for the token definitions in Section 3; class ordering enforced by `prettier-plugin-tailwindcss` rather than left to manual discipline; no arbitrary value classes (`w-[137px]`) outside of the design token scale without an explicit code comment explaining why the scale did not fit.

**shadcn/ui usage**: components are copied into `components/ui/` and owned by the codebase, per shadcn's own model, meaning any customization happens by editing the component file directly, never by fighting the library's props from the outside with wrapper components.

**Best practices**: colocate a component's Storybook story (if used) and test file beside the component, not in a mirrored `__tests__` tree; every domain component in Section 6 ships with at least a loading, empty (where applicable), and error visual state, since those three states are exactly where AI product UIs most commonly ship an unfinished experience.

**Definition of done**: a page or component is done when it has a defined loading state, a defined empty state (if it can be empty), a defined error state, meets the contrast requirements in Section 14, is keyboard-navigable, and its data fetching is fully typed against the generated API types with no manually-written response interfaces. A component that renders correctly with mock data but has not had these five things considered is not done, it is a prototype.