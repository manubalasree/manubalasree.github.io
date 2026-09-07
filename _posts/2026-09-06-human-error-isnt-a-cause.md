---
title: "Why \"Human Error\" Isn't a Cause: It's Where the Investigation Stopped"
date: 2026-09-06
categories:
  - incident-analysis
tags:
  - incident-response
  - postmortems
  - sre
  - human-factors
---

Early in my career as a young sysadmin, I accidentally deleted a customer database.

We had backups. We had a 5-minute RPO. We restored the data quickly and the customer never noticed. By any operational measure, the incident was contained.

The response from my manager was simple: be more mindful of production systems.

I nodded. I agreed. I moved on.

What nobody asked was the question that actually mattered: what enabled a junior engineer to access and modify a production database without any change control in place? There was no break-glass procedure. There was no approval gate. There was no guardrail between my inexperience and a live customer system. The conditions for that deletion existed long before I sat down at that terminal.

I did not know it at the time, but I had just lived my first encounter with what Sidney Dekker calls the Old View of human error. Find the person. Label it carelessness. Move on. The system remains exactly as it was.

I have been in SRE and infrastructure roles for 18 years since that day. I have run postmortems, sat on incident bridges, written action items, and watched engineers get blamed for failures that the system itself made almost inevitable. It took reading Dekker's *Field Guide to Understanding Human Error* to give me the language for what I had been watching all along.

This isn't an academic review. It is six lessons I carried out of that book and back into my daily work.

---

## The label "human error" is where understanding stops, not where it starts

After the database incident, the official conclusion was operator error. I was careless. Case closed.

Dekker's first and most important argument is that this conclusion is not an explanation. It is a label. And the moment you apply that label, you stop asking the questions that would actually make your system safer.

He describes two ways of looking at failure. The Old View sees humans as the problem. Unreliable people undermine otherwise safe systems, so the fix is to find them, retrain them, discipline them, or replace them. The New View sees humans as the symptom. When someone makes an error, the question is not who failed but what in the system made that failure almost rational given what they knew, could see, and were being asked to do at that moment.

He calls this local rationality. Nobody comes to work to do a bad job. The engineer who ran that deployment, the on-call who missed that alert, the sysadmin who deleted that database, they were all acting in ways that made sense to them given the information and pressure they had at the time. Understanding why their actions made sense is where the real investigation begins.

I have seen the Old View operate at every level. Junior engineers told to be more careful. On-call engineers blamed for not following a procedure nobody had tested in two years. Teams reprimanded for incidents that were downstream consequences of architectural decisions made years before they joined. The label feels like a fix. The system stays the same.

There is a concrete diagnostic test for this: look at your postmortem drafts. Whenever you see the phrase 'the engineer failed to' strike it out. That phrase is the Old View announcing itself. Replace it with a question: what made this action make sense to them at the time? The answer to that question is your actual finding.

---

## Blame shifts the goal of incident response from recovery to self-protection

A change gets opened, scheduled, and delegated under significant time pressure. The engineer who receives the task implements it but never updates the change record. When the service degrades, the blame lands on that engineer for not following process.

The truth is usually more complicated. Parallel deployments are running under pressure that should have prevented any single engineer from also managing their own change records. The SRE is also the task manager. The golden rule of keeping SRE task load at a sustainable level had quietly been abandoned under mounting delivery pressure. The conditions for that slip existed across the whole system, not inside one person.

What Dekker identifies, and what plays out on incident bridges across the industry, is that blame does not just feel bad. It actively corrupts the incident response itself. In an environment where pointing at a person is the expected outcome, the goal of everyone in the room shifts. It stops being "how do we restore the service and understand what happened" and becomes "how do I demonstrate that this was not my area, not my change, not my call."

Engineers triage carefully not to fix the problem fastest but to establish distance from it before the postmortem. We have all been in that situation at some point. That is not a character flaw. It is a rational response to an environment where the postmortem is a blame allocation exercise rather than a learning one.

Dekker's point is that you cannot have both. An environment that allocates blame and an environment that generates honest information about what actually happened are mutually exclusive. You have to choose which one your postmortem is for.

---

## Documentation that exists but cannot be reached under pressure does not exist

A service goes down. The incident is picked up by an L2 engineer. Resolving it requires digging through observability logs, traversing to the actual payload sent to a downstream system, and cross-referencing behavior that is only explainable with deep product knowledge built over months.

In the standup after, the argument is straightforward: this is already documented.

It is. Somewhere. In a runbook that was accurate when it was written, in a wiki page three levels deep, in the tribal knowledge of the senior engineer who is not on call that night. With a 15-minute SLA and an evolving incident, the L2 has no realistic path to find that information, comprehend it, and act on it in time. The documentation exists. Under real operating conditions it does not.

Dekker calls this the gap between work as imagined and work as done. Procedures and runbooks describe how work is supposed to happen. Actual work at the sharp end adapts constantly to time pressure, incomplete information, and systems behaving in ways the documentation never anticipated. The gap between those two things is not a sign that engineers are cutting corners. It is where most safety issues actually live.

An enterprise wiki can feel like a collection of outdated documents that made sense to someone at some point and have not been touched since. Just having the information somewhere does not mean it will be usable at the right time. What matters is not whether knowledge exists in the system but whether it can be retrieved and applied by the person who needs it, in the conditions they are actually working in.

The real question to ask after an outage isn’t whether a runbook existed—it’s whether a responder at 2:00 AM could actually locate and execute it under fire.

---

## Low incident numbers are not evidence of a good safety culture

A team is managing an application with a known observability gap. Transient alerts are firing regularly. The team identifies them, documents them as known issues, and the incident count stays low. Leadership sees stable numbers. The system looks healthy.

The real issue is a missed audit log ingestion problem, hidden behind alert thresholds and identifiers that were tuned to reduce noise. In practice those thresholds were filtering out a silent failure that had been accumulating. Missing a single audit log ingestion is not a transient nuisance. In a regulated environment it is unacceptable. But the metric said everything was fine.

Dekker describes this as the decoy phenomenon. A team becomes so focused on the visible, countable, reportable detail that they stop seeing the critical problem developing underneath it. The issues they are tracking are real. But they are outliers that show how the team adapts. The thing that will actually affect them is the one nobody is looking at because it does not show up in the dashboard.

The broader pattern is what he calls the low incident number trap. Observability tools, auditing processes, and incident tracking systems can function as social controls that connect individual behavior to organisational policy. When those numbers get tied to performance reviews, team reputation, or leadership confidence, the incentive inevitably shifts from understanding system resilience to managing appearances. Low numbers get produced. They just do not mean what everyone assumes they mean.

A strong safety culture and a low incident count are not the same thing. Sometimes they correlate. Often they do not.

---

## Procedure is a resource that requires cognitive input, not a guarantee of safety

A deployment pipeline has a codeowner approval process. It has been running for months. The approvers have seen hundreds of PRs come through. They know the pattern. They approve quickly because the process has always worked and they have other things demanding their attention.

Then something slips through.

The Old View response is predictable: the approvers did not check carefully enough. Tighten the process. Add another gate. Require a second reviewer. The assumption underneath all of it is that if the procedure is followed, safety follows automatically.

Dekker proposes something more uncomfortable. Procedure is not a tool that guarantees safety. It is a resource that still requires cognitive input from the people using it. A checklist does not think. A runbook does not adapt. An approval gate does not understand context. The people interacting with those procedures are the ones providing the intelligence that makes them work, and that intelligence degrades under familiarity, fatigue, and time pressure.

This is what makes the SRE on-call environment particularly exposed. Fatigue is structural, not occasional. Familiarity with systems builds over time and creates exactly the conditions where cognitive fixation becomes likely. The team that has handled a hundred deployments without incident is also the team most likely to miss the hundred and first because the pattern recognition that makes them fast also makes them selectively blind.

This dynamic becomes sharper as automated diagnostic tools and AI assistants enter the SRE workflow. These tools accelerate triage, surface anomalies, and draft remediations quickly, but they introduce a classic resilience trap: automation complacency. When tooling appears capable, human attention naturally drifts, yet the requirement for expert judgment actually increases. Someone still has to verify whether the diagnostic output makes sense, catch where it hallucinated or failed subtly, and understand context the system cannot see. Treating automation as a substitute for operational understanding—rather than an amplifier of it—simply adds another rigid layer of procedure while draining the human expertise needed to supervise it. The intuition of an engineer who can read a distributed trace and instinctively smell where state broke cannot be automated away; it is the ultimate safety mechanism.

The implication is not that procedures are useless. It is that procedure plus attention is the actual safety mechanism. When attention degrades, the procedure is running on empty. Investing in follow-the-sun models, sustainable on-call load, and tight integration between observability and runbooks is what keeps that critical cognitive reserve from running completely dry.

---

## The quick fix feels like action but it relocates the problem

Dekker has a line in chapter 8 that is difficult to forget. Reprimanding a bad apple, he writes, is like peeing in your pants. You feel relieved and warm for a little while. Then it gets cold and uncomfortable. And you look like a fool. It is an uncomfortable analogy precisely because most engineers have watched it play out in exactly that sequence.

The pattern repeats across the industry with remarkable consistency. An incident happens. A postmortem is run. Five action items are created. Two get closed. The system drifts back toward the same failure mode and eighteen months later a different engineer in a different timezone triggers the same underlying condition under different circumstances and everyone is surprised.

Dekker calls this the fallacy of the quick fix. Reprimand the person. Retrain the team. Write a new procedure. Reevaluate the technology. These responses feel like action. They produce a paper trail. They satisfy the organisational need to demonstrate that something was done. But they address the visible surface of the incident while leaving the underlying conditions untouched.

What Dekker points toward instead is the slower, harder work of asking what systemic conditions made this incident not just possible but probable. That means looking at goal conflicts, workload distribution, the gap between documented process and actual practice, and the organisational pressures that push engineers toward the sharp end without the authority or information they need to operate safely there.

Quick fixes feel like solutions because they are fast and legible. Systemic interventions feel uncertain because they are slow and their effects are hard to measure. The organisational pressure will almost always favour the former. Knowing that pressure exists and naming it when it appears is the first step toward resisting it.

---

## What the New View actually looks like in an SRE context

![Old View vs New View](/assets/images/posts/oldvsnew.png)

The New View is not a philosophy. It is a set of practical shifts in how you read an incident, run a postmortem, and build the conditions that make your team safer over time.

**On the incident bridge:** The New View asks you to reconstruct what the people involved knew and could see at each decision point, not what you know now with the benefit of hindsight. The question is not what they should have done. It is what made their actions rational given the information and pressure they had at that moment.

**Writing the postmortem:** The New View asks you to treat the human at the centre of the incident as a data point, not an epicentre. Stop at the point where you explain why their actions were logical, then trace backward into systemic design and organizational pressure.

**Evaluating safety metrics:** Separate low incident numbers from real safety. A culture where bad news travels fast, where near misses get reported without fear, and where postmortems generate honest information rather than careful positioning is a safer culture than one with a clean dashboard and silent engineers.

**Reviewing action items:** Ask one question: does this address the condition that made the incident probable, or does it address the most visible part of the incident that leadership needs to see actioned? Both can look identical from the outside. Only one of them changes anything.

None of this is easy in organisations that are built around the Old View. The pressure to find the cause, name the person, and demonstrate action is real and constant. Dekker does not pretend otherwise. What he offers instead is a framework for recognising that pressure when it appears and understanding what it costs when you give in to it.

---

## Closing

Dekker makes a blunt observation in his final chapter: every organisation has room to improve its safety. What separates a strong safety culture from a weak one is not how large this room is. What matters is the organisation's willingness to explore this space, to find leverage points to learn and improve.

After 18 years in SRE and infrastructure roles, that is the most honest thing I have read about safety culture. Not a maturity model. Not a framework. Just a question about willingness.

The Old View is easier. It is faster, more legible, and it satisfies the organisational need for a conclusion. The New View is slower, less comfortable, and it tends to reveal more problems the deeper you look. That is not a reason to avoid it. That is exactly the point.
