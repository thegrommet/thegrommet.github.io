---
layout: post
title: "Estimates: A New Point System"
---

At The Grommet, we use a process derived from Scrum to manage expectations and requests from outside the engineering, operations and design teams. There's a bit of ceremony that isn't too relevant to this post, but one of the practices is estimating the size of tasks as we're planning an iteration of our work (we use two week sprints at the moment). For context, we have seven developers, two product managers, a project manager / Scrum Master, and a company of 80 people in total, supporting an e-commerce site that launches as many as a dozen new products, often entirely new to market each week, including complete media production for then.

We've used several common methods to do these estimates. 

- A weekly planning meeting where we estimate tasks just before assigning them out for the sprint, with a 'planning poker' style exercise where all the engineers opine about the size using unitless numbers, based on the size of similar tasks.
- Senior engineers do a planning poker style exercise during a pre-planning or 'sprint grooming' meeting.
- Senior engineers review a task list and estimate the task using those same numbers
- Senior engineers review a task list and estimate a number of hours, and relate them to the size using a rough "one point is a half day of work" system.

In all cases, we found that estimates were consistently off the mark. Among the problems we experienced:

- 1-point tasks varied greatly in actual time required to execute, from trivial five minute fixes to multi-hour projects with ballooning scopes.
- We didn't always recognize the effect on the rest of the team, the coordination to execute, or the vagueness of the request.
- A task's estimate often didn't correlate to hours because of external factors: stakeholder review steps, code review change requests, discovered problems on the way to implementation, and changing requirements discovered late in the process.
- We estimated bug tickets similarly, but found that we were further off the mark for them.
- We often still correlated point sizes to hours, without evaluating the meta-work of chasing down stakeholders, confirming requests, and iterating through requested changes.

Since I joined the team, I'd been advocating for time to do a verbal "sketch implementation" of each task, talking through what we'd change about each part of our system to achieve the goal, and come up with a rough plan before beginning, and to spend time investigating what would be required before estimating any task that we couldn't do that with.  While there was no resistance to this, there was no formal process for doing so and we routinely skipped this during planning meetings, where everyone was on the linbe.

At another company I'd worked with, we used a simplified system, fully admitting that task descriptions often have no strict relationship to hours worked. We estimated on a three point scale:

- 1 point if the task can be completely verbalized in detail, how it will be performed known exactly
- 2 points if the task can be verbalized grossly, noting which systems will be involved and roughtly what each will have to accommodate
- 3 points if the implementation could not be verbalized

What we discovered there was that all 3 point tasks needed refinement of their specification, and sub-tasks that can be performed first or independently separated, and then the remainder reworked until it could be estimated. We routinely found that inside each 3 point task were multiple other 2 and 3 point tasks, and that it was too large to successfully plan as it was written. Breaking them up gave vastly better planning information, and got the team agreeing on how these requirements would be met.

We are adopting a hybrid of the Scrum and 3-point systems at The Grommet.

For each task, we rate it on four factors:

- _risk_, whether it touches critical systems or revenue generating pathways, or could cause an outage
- _size_, the estimate of how much work the task is (this is basically the old Scrum-style number)
- _complexity_, which we usually measure by listing test cases. More test cases means more complexity
- _uncertainty_, how sure we are of both the written request and the rest of the estimate

All of these we estimate using unitless numbers, trying to relate them to similarly rated tasks in the backlog and previous work. We sum them, and then round up to the nearest fibonacci number.

At the moment we're not rating bug tickets, but I think this schema would work better for bug tickets (with extra points for uncertainty handling some of the uncertain nature of bug investigation)

This has had several interesting effects in our team.

- We discuss implementation plans naturally, because we can't honestly break out a rating without it, and we rely less on how big the task feels like it "should" be.
- We capture the unexpected change in scope by looking at the uncertainty itself, separate from other factors.
- We capture the surprise additional time when we go to deploy or test a change, by evaluating risk and complexity measures separately from the work.
- We notice complexity by explicitly verbalizing testing as a practice, and this accounts better for time to do QA on a change, and surfacing this early often begets simplifications by product team members.
- Verbalizing risk estimates in the presence of product owners and management focuses back the trade-offs between change and risk, and has changed what choices we collectively make to be more valuable to the business and product teams.
- As a side effect of simply summing additional factors onto existing estimates fixed the minimum size of our tasks to make room for a new class of truly trivial tickets.
- The rounding up to the nearest number adds a little breathing room that helps absorb the weekly overhead a bit, so we stay healthy and happy and not overloaded.

With this change, our estimates have gotten much more accurate, and that has allowed our product and project management team to start horse-trading tasks, not externally committing to things we cannot achieve, and to do the hard work of choosing between requests and weighing their relative priority. With this information, they can not overload the engineers with impossible workloads, and they can return to the requesters of the tasks and request more team members be hired, or have the hard conversations about which business priorities they can meet with the time the team has available.
