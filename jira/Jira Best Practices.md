# Best Practices Guide: Vertical Slices in Jira

This guide provides clear instructions, practical examples, and conventions for creating well-structured, traceable Epics, Stories, and Tasks in Jira. The goal is to ensure clear visibility, accurate tracking, and improved productivity.

## 1. What is a Vertical Slice?

A vertical slice is a small, self-contained increment of work that delivers a complete end-to-end feature. This includes UI, backend/API, and database changes.

## 2. Structure & Naming Conventions

**Epic:** Broad but clearly scoped feature or capability.

* Format: `Epic - [Feature Area] Brief Description`
* Example: `Epic - User Authentication Management`

**Story:** Specific, user-focused requirement within an Epic.

* Format: `[Area][Feature] User-focused story description`
* Example: `[Auth][Login] User login with email/password`

**Sub-task:** Technical implementation tasks within a story.

* Format: `[Area][Tech Layer] Short Technical Description`
* Example: `[Auth][API] Create login authentication endpoint`

## 3. Practical Example

### Epic

* **Epic - Article Management System**

### Story

* `[Articles][Publishing] Publish/Archive Article`

**Acceptance Criteria:**

* Users can publish or archive articles.
* Archiving an article archives references and comments.

### Sub-tasks

* `[Articles][UI] Implement Publish/Archive UI controls`
* `[Articles][API] Endpoint for Publish/Archive logic`
* `[Articles][DB] Update schema for publication statuses`

## 4. Writing Good Acceptance Criteria

Clearly defined acceptance criteria set the conditions for completion:

* Keep it concise and clear.
* Bullet points for each specific condition.
* Example:

  * ✅ User can search articles by keyword.
  * ✅ Results limited by user's permissions.
  * ✅ Response time <500ms.

## 5. Using Jira Automation

Automate repetitive Jira setup tasks:

* Automatically create default sub-tasks for stories.
* Example automation:

  * When creating a `[Auth][Login]` story, Jira auto-creates:

    * `[Auth][UI] Login Form`
    * `[Auth][API] Login Endpoint`
    * `[Auth][DB] User Schema`

## 6. Do's and Don'ts

**✅ Do:**

* Clearly link sub-tasks to stories and stories to Epics.
* Ensure each vertical slice can be independently tested.
* Use labels or components for easy filtering (e.g., `UI`, `API`, `DB`).

**❌ Don't:**

* Create overly large Epics or vague stories.
* Skip acceptance criteria.
* Leave sub-tasks unassigned or poorly defined.

## 7. Regular Review and Improvement

* Weekly backlog grooming to maintain Jira hygiene.
* Regular retrospectives to discuss vertical slicing effectiveness.

## 8. Example Jira Structure (Visual Reference)

```
Epic - Article Management System
│
├── STORY [Articles][Search] Article Search and Filter
│   ├── SUBTASK [Articles][UI] Build Search Interface
│   ├── SUBTASK [Articles][API] Article Search Endpoint
│   └── SUBTASK [Articles][DB] Optimize Search Query
│
└── STORY [Articles][Publishing] Publish/Archive Article
    ├── SUBTASK [Articles][UI] Publish/Archive UI
    ├── SUBTASK [Articles][API] Publishing Logic
    └── SUBTASK [Articles][DB] Publication Schema
```

Follow this guide consistently to ensure clarity, traceability, and efficiency across your team.

