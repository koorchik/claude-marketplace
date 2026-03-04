---
name: analyze-frontend-architecture
argument-hint: <path-to-repository>
disable-model-invocation: true
user-invocable: true
description: Analyze React frontend architecture for component design quality, data flow clarity, and Single Level of Abstraction violations.
---

# Analyze Frontend Architecture

You are performing an architecture review of a React frontend project. Your goal is to produce a clear, actionable report that helps the developer improve their component design, data flow, and code organization.

## Philosophy

This review is grounded in four principles that reinforce each other:

1. **Single Level of Abstraction (SLA)**: Each component should operate at one consistent abstraction level. A page component composes high-level domain components — it should not mix `<OrderSummary>` with raw `<div className="flex gap-2">`. When you see a component mixing abstraction levels, that's the most important signal something needs restructuring.

2. **Clear Data Flow with API Abstraction**: Every component should have a well-defined API (props in, callbacks out). Data should flow predictably — parent owns state, children receive what they need, callbacks bubble events up. Critically, there should be an **API abstraction layer** between the application and external services. Hooks and components should never call Firebase, fetch, axios, or any external API directly. Instead, a **service class** (e.g., `api/ApplicationsApi.ts` or `services/ApplicationService.ts`) should own all external communication, and hooks should be thin wrappers that manage React state on top of these services. The service should be a class — not bare exported functions — because real API layers have dependencies that need constructor injection (base URL, auth tokens, HTTP client instance, timeout config). A class makes these dependencies explicit and testable: `new ApplicationsApi({ baseUrl, token })` in production vs `new ApplicationsApi({ baseUrl: 'http://mock' })` in tests. This layering means backends can be swapped (Firebase → REST → GraphQL) without touching any hook or component.

3. **Minimal Duplication**: Similar patterns repeated across components signal a missing abstraction. But extract only when the duplication is structural (same props pattern, same composition shape), not merely visual. When multiple hooks duplicate the same external API calls, the fix is NOT to merge them into one mega-hook — it's to extract the shared logic into an API service layer that both hooks consume.

4. **Proper Layering**: The architecture should follow clear layers: **API/Service layer** (owns external communication) → **Hooks** (own React state, consume API layer) → **Components** (own rendering, consume hooks). When you see hooks directly importing Firebase SDK, HTTP clients, or other external libraries, that's a layering violation. Each layer should depend only on the layer below it.

## Analysis Workflow

### Step 1: Map the project structure

Use Glob and Read to understand the project layout:
- Find all component files (`**/*.tsx`, `**/*.jsx`)
- Identify page-level components (usually in `pages/`, `routes/`, `views/`, or `app/` directories, or files with "Page" / "Screen" / "View" in the name)
- Identify shared/reusable components (usually in `components/`, `ui/`, `shared/`)
- Identify hooks (`**/use*.ts`, `**/use*.tsx`)
- Look for state management (stores, contexts, reducers)

Build a mental map: pages → features → shared components → hooks → state.

### Step 2: Analyze page-level components for SLA

For each page component, check:
- Does it compose high-level domain components at a consistent abstraction level?
- Or does it mix domain components with raw HTML elements, inline styles, or low-level logic?
- Are there blocks of JSX that could be named (extracted into a component with a descriptive name)?

**What to look for:**
- A `<DashboardPage>` that has `<StatsPanel>` next to a raw `<div>` with 15 lines of markup — SLA violation
- Inline state transformations or data formatting inside JSX — should be in a hook or utility
- Conditional rendering logic (ternaries, `&&` chains) that obscures the page's high-level structure

### Step 3: Analyze component APIs (props and callbacks)

For each component, evaluate:
- **Props clarity**: Are props well-named and minimal? Does the component receive only what it needs?
- **Callback design**: Are callbacks properly typed? Do they expose the right level of detail? (e.g., `onSelect(item)` is better than `onClick(event)` when the parent cares about the item, not the click)
- **Prop drilling**: Are props passed through 3+ levels without being used in intermediate components? This signals missing context or composition restructuring.
- **God components**: Components with 10+ props often do too much — check if they should be split.
- **Boolean prop explosion**: Multiple boolean props like `isLoading`, `isError`, `isEmpty`, `isDisabled` often signal that the component should accept a status/state union instead.

### Step 4: Analyze composition patterns and Layout

Look for:
- **Children vs props**: Is `children` used effectively for composition, or are components overly configured through props?
- **Render props / function-as-children**: Are these used where simpler composition would work?
- **Component responsibility**: Does each component have a single, clear job?
- **Wrapper hell**: Deeply nested providers/wrappers that could be flattened.

**Check for a shared Layout component** — this is a common and impactful missing abstraction:
- Do multiple pages independently render the same header, navigation, or footer?
- Is there a `Layout`, `AppShell`, or `MainLayout` component that provides the application shell?
- If pages each render their own `<PageHeader>` with hardcoded navigation arrays, that's duplicated app-level structure that should be centralized into a Layout component.

**The Layout must be composed inside each Page component, not at the router level.** This is a deliberate architectural choice:
- **Framework-agnostic**: Works identically with React Router, Next.js, Remix, or any routing. No dependency on `<Outlet />` or router-specific APIs.
- **Configurable**: Pages can pass config props (title, subtitle) to customize the shell. For advanced cases, pages can pass JSX to named slots (sidebar, footer) — something router-level layouts make awkward.
- **Self-contained pages**: Each page is a complete, end-to-end unit. Reading a page file shows you everything it renders, including its shell.
- **Routing independence**: Changing routes, adding nested routes, or restructuring the router never affects page layout.

**Example of what to recommend for Layout:**
- BAD — Layout at the router level using `<Outlet />`:
  ```tsx
  // App.tsx — couples layout to React Router
  <Route element={<AppLayout><Outlet /></AppLayout>}>
    <Route path="/" element={<HomePage />} />
    <Route path="/settings" element={<SettingsPage />} />
  </Route>
  ```
  This ties layout to React Router specifically, makes pages incomplete fragments that only work inside an Outlet, and makes it hard for individual pages to customize their shell (e.g., a settings page that needs a sidebar). Do not recommend this pattern.

- GOOD — Each page composes the Layout itself:
  ```tsx
  // components/AppLayout.tsx — owns ALL layout markup (header, nav, footer)
  export default function AppLayout({ title, subtitle, children }: Props) {
    return (
      <div className="app-layout">
        <PageHeader title={title} subtitle={subtitle} nav={APP_NAV} />
        <main>{children}</main>
      </div>
    );
  }

  // pages/HomePage.tsx — self-contained, complete page
  export default function HomePage() {
    return (
      <AppLayout title="Dashboard">
        <HomeOverview stats={stats} />
        <RecentActivity items={items} />
      </AppLayout>
    );
  }
  ```
  By default, `AppLayout` should own all layout markup (header, nav, footer) internally and accept only configuration props like `title`. Pages pass simple config, not full JSX components. This keeps `AppLayout` as the single source of truth for the application shell. Passing JSX as slot props (e.g., `header={<Custom />}`) is acceptable for advanced cases but should not be the default recommendation. No router dependency — works with any framework.
### Step 5: Analyze data flow and API abstraction

Trace how data moves through the app:
- Where does state live? Is it as close as possible to where it's used?
- Are there cases where global state is used for what should be local state?
- Do components fetch their own data, or is data fetching centralized at the page level?
- Are there circular dependencies or unclear data ownership?

**Check for API abstraction layer** — this is critical:
- Do hooks or components import external SDKs directly (Firebase, axios, fetch, Supabase, etc.)?
- Is there a dedicated `api/`, `services/`, or `lib/` directory that wraps all external calls?
- If multiple hooks call the same external API, the fix is to create a shared service layer — NOT to merge hooks into one large hook. Each hook should have a clear, focused responsibility; the API layer handles the shared external communication.

**API layer should be a class, not bare functions.** A class allows constructor injection of dependencies like base URL, auth tokens, and HTTP client — making it configurable and testable.

**Example of what to recommend:**
- BAD: `useApplications.ts` imports `firebase/firestore` and calls `addDoc`, `onSnapshot` directly
- GOOD: `api/ApplicationsApi.ts` exports a class:
  ```tsx
  class ApplicationsApi {
    constructor(private config: { db: Firestore }) {}
    subscribe(cb: (apps: Application[]) => void) { /* onSnapshot */ }
    async create(companyId: string, role: RoleType) { /* addDoc */ }
    async remove(id: string) { /* deleteDoc */ }
    async updateStatus(id: string, status: ApplicationStatus) { /* updateDoc */ }
  }
  ```
  The hook instantiates or receives the API class and manages React state on top of it.
- This means if the team migrates from Firebase to a REST API, they only change the service class, not every hook. And in tests, they can pass a mock config or mock the class entirely.

### Step 6: Identify code duplication

Look for:
- Components that are nearly identical but with small variations (candidates for a shared component with props)
- Repeated patterns of hooks + state + effect that could be a custom hook
- Similar data transformations done in multiple places

## Output Format

Write the report to `ARCHITECTURE_REVIEW_RESULTS.md` in the root of the analyzed project.

### Report Structure

```markdown
# Architecture Review Results

> Analyzed on: [date]
> Project: [name/path]
> Total components analyzed: [N]
> Issues found: [N]

## Summary

[2-3 sentence overview of the project's architecture health. Lead with the most impactful finding.]

## Issues

### [ISSUE-ID]: [Short descriptive title]

**Severity**: High | Medium | Low
**Principle**: SLA Violation | Unclear Data Flow | Missing API Abstraction | Missing Layout | Code Duplication | Poor Component API
**Location**: `path/to/file.tsx`

[1-2 sentence explanation of WHY this is a problem — what breaks, what becomes harder to maintain, what confuses readers.]

#### Current (Bad)

` ` `tsx
// Shortened version of the actual code showing the problem
` ` `

#### Recommended (Good)

` ` `tsx
// How it should look after fixing
` ` `

**Why this is better**: [1 sentence explaining the improvement]

---

[Repeat for each issue]

## Recommendations Summary

| Priority | Issue | Effort | Impact |
|----------|-------|--------|--------|
| 1 | [title] | Low/Med/High | Low/Med/High |
| 2 | ... | ... | ... |

## Architecture Health Score

| Criterion | Score (1-5) | Notes |
|-----------|-------------|-------|
| Single Level of Abstraction | | |
| Component API Design | | |
| Data Flow Clarity | | |
| API Abstraction Layer | | |
| App Layout / Shell | | |
| Code Duplication | | |
| Composition Patterns | | |
| **Overall** | | |
```

### Writing the Issue Examples

The CURRENT (Bad) and Recommended (Good) examples are the most important part of the report. Follow these rules:

- **Keep examples short** — 5-15 lines each. Show the essence of the problem, not the entire file. Use `// ...` to skip irrelevant parts.
- **Show real code from the project** — do not invent generic examples. The developer should recognize their own code.
- **The Good example must be realistic** — it should compile (conceptually) and respect the project's existing patterns. Do not introduce new libraries or paradigms the project doesn't use.
- **Name things well in the Good example** — component names in the fix should communicate intent. If you extract a component, give it a name that makes the page-level composition read like a sentence.
- **One issue per example** — do not try to fix multiple problems in a single Good example. If a component has both SLA and prop issues, create separate issue entries.

### Severity Guidelines

- **High**: Affects multiple components, makes the codebase actively harder to work with. SLA violations in page components, widespread prop drilling, duplicated stateful logic.
- **Medium**: Localized problem that affects readability or maintainability of a specific area. Oversized component APIs, inconsistent callback patterns.
- **Low**: Style/organization preference that would improve clarity. Minor naming issues, slightly suboptimal composition.

### Tone

Be direct and constructive. The report should read like advice from a senior colleague — specific, grounded in the actual code, and focused on the "why" behind each recommendation. Avoid vague statements like "consider improving" — say what to do and why.

## Important Notes

- Analyze ALL page-level components, not just a sample. Pages are where architecture problems are most visible.
- Do not comment on CSS, styling, or visual design — this review is purely about component structure and data flow.
- Do not suggest adding libraries or changing the tech stack unless the existing patterns are fundamentally broken.
- Prioritize issues by impact — lead with the findings that would make the biggest difference if fixed.
- If the project is well-architected, say so! Not every review needs to find problems. A short report that says "this is solid, here are two minor suggestions" is perfectly valid.
- When recommending a Layout component, always show it composed inside Page components (each page wraps its content in `<AppLayout>`). Never recommend `<Outlet />` or router-level layout wrappers — the Layout-inside-Page pattern is framework-agnostic and keeps pages self-contained.
