# 📄 Reflections from a Fresh Pair of Eyes

**Prepared by:** \[Your Name]
**Role:** Lead Developer
**Audience:** Planning & Management Group
**Date:** \[Insert Date]
**Purpose:** To share observations on delivery pace, process, and team dynamics from a new perspective, based on recent 1:1s and cross-team collaboration.

---

## 👋 Introduction

Since joining the team, I’ve had the opportunity to observe our delivery flow and speak to many people across disciplines. This document is not a critique — it’s simply a reflection on what I’ve seen and heard, offered as a **friendly, fresh pair of eyes**.

It aims to help us better understand one key question:

> **Why aren’t we delivering at the pace some parts of management expect?**

I’ll offer some hypotheses, patterns, and open questions — along with suggestions that could help us better align delivery with expectation and reality.

To be clear: this is **not a call to overhaul our delivery process overnight**. In fact, changing too much too quickly may obscure what's working and create unintended consequences. Instead, we might consider **small, low-risk improvements**, test them in real delivery, and then reflect on their impact together.

---

## ✅ What’s Working Well

* **UI Design Walkthroughs**: Once complete, they create excellent shared understanding.
* **Spikes**: Highly effective at surfacing complexity early.
* **UI Team Collaboration**: Strong cohesion and problem-sharing via Slack.
* **Code Quality**: App B2’s frontend is cited as particularly well-structured.
* **UI Tickets**: Generally well-scoped and easier to pick up compared to backend stories.

---

## 🐒 Where Delivery Slows Down

### 1. **Stories Start Too Early**

* Stories are often written and created in bulk after requirements gathering, resulting in a mesh of dependencies and assumptions. Rather than work starting too early, it often starts too rigidly — with little room for teams to reshape or challenge what's been handed down.
* Leads to rework, frustration, and slow progress mid-sprint.

### 2. **Epics Run for Months**

* Long-running epics blur focus and make meaningful progress difficult to demonstrate.

### 3. **Shared API = Shared Risk**

* Four apps use the same backend with no formal API ownership or governance.
* One change can block or break others.

### 4. **Blockers Surface Late**

* Developers don’t always raise blockers early — sometimes due to confidence or context gaps.
* This causes hidden delays and late surprises.

---

## 🔍 Themes from 1:1 Conversations

These are consistent messages that came through from multiple team members:

### 🔄 Working Practices

* “We don’t do Agile. It’s Waterfall with Sprints.”
* Stories are prewritten after requirement workshops — *not* iteratively refined.
* UI often blocked by late backend alignment.
* Design sign-off delays start, but there’s no feedback loop once work begins.

### 🧱 Process & Tools

* **Confluence** is considered overloaded, underused, and disconnected from actual delivery.
* Preference is for **README.md in-repo** and developer-proximate documentation.
* **Spikes** are working well technically but not always understood or valued by business stakeholders.

### ⚙️ Ownership

* **API is treated as an afterthought**, but should be a product in its own right.
* No defined guardianship for API changes or quality.
* Developers want a **contract-first or vertical-slice approach** to cross-team work.

### 🧠 Skills, Risk & the Role of a Lead Developer

* Team skill levels vary — newer team members sometimes get blocked and don’t escalate.
* Much of the system knowledge sits with contractors — some leaving in **November or February**.
* Pairing and mentoring are needed but have delivery cost.

As I’ve settled into the Lead Developer role, I’ve come to believe there is an opportunity to better connect the technical and delivery sides of our process. A Lead Developer can serve as a bridge — helping teams surface blockers earlier, clarifying ambiguity between disciplines, and translating evolving requirements into practical implementation strategies. Ideally, I can support this by reinforcing shared context across frontend, backend, and product conversations — without adding unnecessary process.

---

## 🧍️‍♂️ Definition of Ready — Are We Waiting Too Long to Start?

In theory, development starts after requirements gathering and story writing are complete. However, there is often **downtime between these steps** — between initial analysis, understanding, documentation, and actual story readiness. As a result, teams may pick up work that appears ready on the surface, but lacks up-to-date clarity, developer input, or validated context.

We appear to wait until:

* Requirements are final
* Design is approved
* Backend expectations are documented
* Stories are “signed off” by a BA or lead dev

This feels like a **Waterfall mindset**, not Scrum.

> In Scrum, work starts when a story is *“ready enough”* — not perfect. That means:
>
> * The goal and value are clear
> * Acceptance criteria are defined
> * Scope is small and deliverable
> * Dependencies are known (even if not resolved)

But many of our stories are being **pre-baked** before development begins. This creates rigidity, delays, and disconnects between plan and reality.
Often, stories are written **all at once** after the requirements — instead of being **drip-fed** into the backlog as they become clear.

---

## ❓ Reflections & Open Questions

These aren’t assertions — just things I’ve noticed and would like to explore with the group:

* **Where does the project *actually* begin?**
  First Confluence page = March 3rd.
  Slack channel created = June 6th.
  → Do we all share the same start date?

* **Is there a confidence gap in shaping stories?**
  Are analysts or leads waiting too long to define “safe” stories?
  → Would earlier team involvement improve shared confidence?

* **Are we writing too much, too early?**
  → Is front-loading all stories helping or hurting iteration and adaptability?

* **Is our delivery process truly iterative?**
  → Or is the appearance of agility masking a sequential, approval-driven model?

* **Are blockers and complexity being surfaced early enough?**
  → Could we raise and track blockers more transparently before they become critical?

---

## 🚀 Opportunities to Explore

These are **not mandates** — just ideas that could improve clarity, flow, and alignment — ideally trialled in small ways before being scaled up:

| Opportunity                                             | Why It Might Help                                   |
| ------------------------------------------------------- | --------------------------------------------------- |
| **Agree on a lightweight Definition of Ready**          | Prevents premature starts and rework                |
| **Introduce pairing or shadowing for critical stories** | Upskills team and reduces single points of failure  |
| **Nominate API ownership / stewardship**                | Makes shared backend changes more predictable       |
| **Create “contract stories” for API↔UI collaboration**  | Clarifies integration boundaries early              |
| **Simplify documentation expectations**                 | Shift to closer-to-code docs and reduce duplication |
| **Make blockers more visible in real-time**             | Enables faster support and prioritisation           |

---

## 🤝 Closing Thoughts

The team is capable and collaborative. The challenges we’re facing are not about people — they’re about **structure, clarity, and alignment**. What we call Scrum isn’t always Scrum, and that’s OK — but we may need to realign what we expect from our process and how we support our teams in delivering value iteratively.

My hope is that we can explore a few of these ideas in small, measurable ways. That way, we can see what works in practice — and course-correct as needed.

Thanks for reading,
**\[Your Name]**

---

## 📌 Appendices (Optional for Review or Breakout Later)

* **Full team feedback from 1:1s**
* **Examples of long-running epics**
* **Summary of blockers raised during last 2 sprints**
* **March–June timeline highlights (Slack vs Confluence vs Delivery)**
