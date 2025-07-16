# 📄 Delivery Reflection – Summary View

**Prepared by:** \[Your Name]
**Role:** Lead Developer
**Audience:** Planning & Management Group
**Date:** \[Insert Date]

---

## 👋 Purpose

As a fresh pair of eyes on the team, I’ve been gathering insights from recent 1:1s, team observations, and delivery experience. This short summary highlights a few key themes and small opportunities for improvement that could help us understand why delivery sometimes moves slower than expected.

This is not a proposal to overhaul the process — just a friendly contribution to support our shared goals. These are not absolutes these are observations, which can be challenge for their accuracy

---

## 🚧 Key Observations

* **Stories are front-loaded and overly rigid?**
  Stories are written in bulk after and accepted into the the development team in back log in bulk, some of these stories have an intricate dependency graph which results in critical paths meaning parralising the work can be difficult. This limits team flexibility and can delay value delivery. It appears we only consider writing stories when we are happy with the UCD and the business analysis. Maybe our Definition of Ready is aa bit to strict?

* **Delivery is slowed by shared backend/API**
  Four apps depend on the same backend without clear API ownership. One change can unintentionally affect others, though the work on this is ongoing, it is and wil continue to dedupe these applications. 

* **Blockers often surface too late**
  Teams sometimes hesitate to raise blockers, or who to raise them to until it’s already affecting delivery — possibly due to unclear ownership or support pathways. Hopefully having a lead developer will alleviate this issue 

* **Documentation is too heavy, too disconnected**
  Developers prefer simple, code-adjacent docs over scattered or outdated Confluence pages. Anything code, or code design based should be part of the documentation of the app. Prefer github markdown over confluence. 

* **Team capability is uneven and at risk**
  Some critical knowledge lives with contractors who may leave later this year. Newer team members sometimes lack confidence to ask or escalate.

* ** Team1/2/3 **
  Communication between the team could be better. there is very much a TeamA / Team,3 divide. This does not result in any nefarious wall building behaviour. People are helpful and collabarative with one another, but there is a divide. The fact that we enforce that divide in the naming of the slack channels perpetuates this

* **Slack Channels**
  There are a lot of slack channels, Being in the management team I can see a lot of channels, and I'm never always so sure why people are in them. Having a slack channel for a feature is a great idea, but perhbaps we should consider just putting all of the devs in there? it would aid communication by osmosis in the team.

---

## What's working well

* UI Design Walkthroughs: create excellent shared understanding.
* Spikes: effective at highlighting complexity early.
* UI Team Collaboration: Strong cohesion and problem-sharing via Slack
* Code Quality: One developer mentioned that **** is the best version of the UI he has worked on, due to it being the newest and excluding non ** choices
* UI Tickets: Generally very clear, very visual and easier to pick up and run with than backend stories

--- 

## 🔄 Role of a Lead Developer

As I grow into the Lead Developer role, my goal is to help bridge the gaps across tech and delivery — by reinforcing context, surfacing blockers earlier, and helping translate evolving needs into shared understanding. I hope to support both consistency and flexibility across the team.

---

## 💡 Opportunities to Explore (Small, Measurable Changes)

| Opportunity                                 | Why It Might Help                                            |
| ------------------------------------------- | ------------------------------------------------------------ |
| **Lightweight Definition of Ready**         | If we are doing scrum, (which I believe we are), we should be creating stories as soon as possible, and marking then as rady with a confidence %. we should be asking qhy can't we strt a ticket. We should be embracing change, if we need a new ticket to fix something after we raise it and schedule it          |
| **API stewardship or ownership**            | Reduces cross-app risk and builds confidence in changes      |
| **Introduce "contract stories" for UI↔API**  | Aligns teams around data flow and boundaries earlier         |
| Need to rethink story structure to prevent prevent tickets not being able to be picked up   |          |
| **Smarter docs**                            | Focus on in-repo, close-to-code clarity                      |
| **Support pairing on critical stories**     | Builds shared knowledge, reduces reliance on few individuals |
| **Make blockers more visible, earlier**     | Supports better intervention and planning                   |
| **Why so many slack channels**     | We have so many channels, do we need all of them, is it not seperating the team? I personally miss information on channles as I am subscribed to just so many, I waste time trying to find the correct channel and then check if that person is in there                 |


---

## Case study ****

What is the perception across the team fo this work starting? 
Confluence Page Feb 26th
Requirement Feb26th - March -24th 
Screen flow ddiagrean finished may 13th 
Slack chaneel June 6th
Design : user sperience, policy approx July 2nd  
Develkopment User Stories: 16th July

Does this allign with everyones understanding

---

## User Story breakdown 

Option 1: Vertical slices
   pros: testible units / FE and BE are tasks in a story, Story should be completed in a sprint, if it can't be its to big, and gets split up into stories with an epic. 
         amendments to the story can be created as new stories
   cons: very different to what we do now, it would mean that all layers of the buisness can can actually  slice their work. This only works if the full team adopts the process. 
Option 2: Contract Story 
   pros: Not such a break away from what whe have to do, but we createa supporting story that contains the DOC the api contract. This story is linked to the backend and the frontend ticket. FE then codes to the contract independently of the backend that aslo codes to the contract, this means that work is not blocked by other work, tickets can be delivered, and the contract story gets scheduled in a sprint once FE and BE is complete. this becomes the ticket that, tests the integration and gets blocked at QA.

---

## 🤝 Final Thought

This is just one view, shared with humility and curiosity. I’m excited to work with you all to explore a few of these ideas in a low-risk, collaborative way.

Thanks for reading,
**\[Your Name]**
