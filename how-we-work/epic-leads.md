# How to be an Epic Lead

## Background

Every quarter, each engineer will typically have at least one Objective that is their responsibility to lead, work on and land during the quarter. Each engineer will work with their manager to figure out what these Objectives are prior to the start of a quarter. Ignoring Technical Objectives for the time being, the Product Objectives' priorities are set be the Product team in the previous quarter based on Product Discovery and User Research and dictate the next set of features to ship. The priority designates the _order_ in which the Objective will be worked.

In order to figure out when you will be called to "batter up" (i.e. become an Epic Lead to start leading the charge on your Objective), you can take a look at the Epic column in our [Zenhub board](https://app.zenhub.com/workspaces/single-cell-5e2a191dad828d52cc78b028/board?repos=105615409,228681195,245246384,280546849,313382406&showPipelineDescriptions=false&showReleases=false) to roughly estimate when your Objective will have resources to be worked on (and here when I say "resource" I mean bandwidth from both Product Management and Engineering).

The subsequent sections of this document describe what happens once it is your turn to be an Epic Lead. One note, at this point, I will start using the term Epic. An Epic is some demoable slice of work. They can be nested. Typically an Objective is its own Epic (thus the terms can be used interchangeably). However, an Objective can be broken down into smaller demoable chunks and each of those demoable chunks is also an Epic.

## The Design Phase

Once an Objective/Epic is ready to be worked on, the Epic enters the "Design Phase." During this phase the end goals are as follows:

<<<<<<< HEAD
- The Objective is broken down into smaller chunks, each of which is a "Key Result" for that epic. These Key Results are written down in the OKR spreadsheet (reach out to your manager to figure out where this spreadsheet is if you don't know). The Product Manager will create child Epics in Zenhub accordingly.
=======
- The Objective is broken down into smaller chunks, each of which is a "Key Result" for that epic. These Key Results are written down in the OKR spreadsheet (reach out to your manager to figure out where this spreadsheet is if you don't know).
- Each of the above smaller chunks has an Epic in Zenhub associated with it. The Product Manager will help you craft these issues.
>>>>>>> First draft of Epic Leads process.
- The top level Epic issue in Zenhub (the one that is associated with the Objective) has links to Figmas (from UX) as well as any Technical Design Document(s) that were created in order to figure out what to build.
- Each Epic is populated with a set of issues that represent the work needed to be done to complete the epic. These issues will be estimated by the full engineering team so please make sure that they contain enough detail that anyone could implement it _and_ please make sure that you have verified with the Product Manager that the issues represent the full set of work to be completed and nothing has been left out.
- Any dependencies between issues have been marked in Zenhub so that we understand which issues need to be worked on first.

<<<<<<< HEAD
### What should I do during the design phase

As an Epic Lead, you have flexibility in the way that you accomplish the goals above, but below are some helpful guidelines. The nucleus of a successful design phase is maintaining a close working relationship between yourself, the Product Manager who is responsible for the product requirements for the Objective, and the UX Designer who is responsible for the visual design (if necessary). As long as you maintain a frequent and open line of communication with those two people (sometimes fewer, sometimes more people will be involved depending on the scope of the Objective), you are on the path to success :)

This phase is really meant to be for you and PM to figure out exactly what will be built for that particular objective. This is an opportunity for you to talk about different build options and figure out what can be built in a _minimal_ yet _substantive_ way. You should be _partnering_ with PM and UX on this and having clarifying discussions on product requirements and the _cost_ (both monetary and effort) of different solutions.

**Recommendation of events that you should schedule/do as an Epic Lead**:

The Design Phase is roughly broken into two parts. The first phase's goal is to **refine the product requirements for the Epic.** During this phase, the following events will likely happen:

- PM will reach out to schedule a meeting to go over the current product requirements along with UX. This is your opportunity to ask clarifying question, offer initial ideas, figure out what open questions remain and along figure out whether there is anything to prototype so that you can get rough cost estimates on different solutions. Think of this as a "negotiation" phase.
- You will likely have follow up meetings after the initial meeting to go back and revisit the open questions/share findings from any research done.
- [Optional but helpful] Create a temporary Slack channel with the PM and Designer (and anyone else who might be involved). Use this channel to resolve questions quickly, keep everyone up to date on progress and coordinate any events/situations that may arise.

One may make use of the Tuesday Refinement meetings (when there are no issues to point) to continue refining the Epic until the design and implementation details are sorted out.

After the product requirements are set, the second part of the design phase is focused on **refining the technical implementation.** During this phase, you may:

- Write a design document. If the Objective is substantial and will involve modification to the system architecture, you should write a design document and get it approved by the engineering team. The details on how to do that are in the [RFC document](../design-documents/0000-rfc-process/text.md).
- Craft a project plan. If the Epic is substantial and there are many moving parts, you may want to create a short document on how you intend to stage the implementation of the child Epics (i.e. when and in what order). This is a way to help manage expectations so that PM might know when to review issues created for a child Epic, when something is planning on being pulled into a sprint, when a technical design doc will land, or even before the technical design phase, when you plan on completing some prototyping work.
- Schedule a meeting with PM+UX to verify that the child issues that you have created for an Epic or child Epic actually implement the agreed upon product requirements (this should happen before the issues get placed into PokerBot for estimation).

The Design Phase of an Epic is squishy and uncertain and so the path to the goals described above can be different each time. If you are struggling on how to find clarity or need any recommendations on next steps, reach out to your peers, ask PM/UX, or check in with your manager.
=======
### What should I do during the design phase?

As an Epic Lead, you have flexibility in the way that you accomplish the goals above, but below are some helpful guidelines. The nucleus of a successful design phase is maintaining a close working relationship between yourself, the Product Manager who is responsibile for the product requirements for the Objective, and the UX Designer who is responsible for the visual design (if necessary). As long as you maintain a frequent and open line of communication with those two people (sometimes fewer, sometimes more people will be involved depending on the scope of the Objective), you are on the path to success :)

This phase is really meant to be for you and PM to figure out exactly what should be built for that particular objective. This is an opportunity for you to talk about different build options and figure out what can be built in a _minimal_ yet _substantive_ way. You should be _partnering_ with PM and UX on this and having clarifying disucssions on product requirements and the _cost_ (both monetary and effort) of different solutions.

**Recommendation of events that you should schedule/do as an Epic Lead**:

- Immediately after your Objective enters the Design Phase, schedule a meeting with the PM and Designer to go through the Objective in detail and figure out what open questions remain. Figure out who will answer what questions and when to regroup to determine a decision and next steps.
- Craft a project plan. Work with the PM to figure out an estimate to how long the Objective will be in the design phase and during what sprint implementation work will begin. Create a working document that you add to over time that you share with PM and UX (and the larger engineering team as a courtesy) to set expectations on when you all will have various parts of the design completed. For example, set a date for when you as the Epic Lead will complete a draft of the issues for the sub-Epics so that PM knows when to verify the issues. If the Objective will require multiple sprints to design, write down when sub-parts of the Objective can be implemented so that unblocked implementation work can begin. Write down dates for any design documents you are writing and when you expect it to be complete. Work with PM to figure out when any follow-up User Research is expected to complete so that you know when you are unblocked on technical design. Write down dates for when you expect to complete any prototyping work so that PM knows when to follow up. The point of this document is not to be "waterfall"-esque but rather to help manage and set expectations on when someone should be ready to go on their next step.
- Write a design document. If the Objective is substantial and will involve modification to the system architecture, you should write a design document and get it approved by the engineering team. The details on how to do that are in the [RFC document](../design-documents/0000-rfc-process/text.md).
- Create a temporary Slack channel with the PM and Designer (and anyone else who might be involved). Use this channel to resolve questions quickly, keep everyone up to date on progress and coordinate any events/situations that may arise.
- Make use of the Tuesday Refinement meetings (when there are no issues to point) to continue refining the Epic until the design and implementation details are sorted out. You will likely need more than the initial meeting to refine the epic so use the Tuesday 10am PST hour as needed.

This phase of an Epic is squishy and uncertain and so the path to the goals described above can be different each time. If you are struggling on how to find clarity or need any recommendations on next steps, reach out to your peers, ask PM/UX on what you can do to help bring clarity or check in with your manager.
>>>>>>> First draft of Epic Leads process.

## Implementation Phase

Hooray! Now that you, PM, and UX have worked tirelessly to bring clarity to the project, now we can work together to bring it fruition. Note here that we work together as an engineering team to implement projects. Why? To reduce knowledge silos. Here are the scrum events you should be watching out for:

### Refinement Meeting -- the Tuesday prior to the start of the sprint

<<<<<<< HEAD
<<<<<<< HEAD
This is the most important refinement session for an Epic Lead. This is the Refinement session that happens prior to pulling in the Epic into the sprint. This is when the tickets you have created will get estimated by the team.

**Before** this refinement, there are the following expectations:

=======
This is the most important refinement session for an Epic Lead.  This is the Refinement session that happens prior to pulling in the Epic into the sprint. This is when the tickets you have created will get estimated by the team. 

**Before** this refinement, there are the following expectations:
>>>>>>> First draft of Epic Leads process.
=======
This is the most important refinement session for an Epic Lead.  This is the Refinement session that happens prior to pulling in the Epic into the sprint. This is when the tickets you have created will get estimated by the team.

**Before** this refinement, there are the following expectations:

>>>>>>> Fix linting issues.
- Tickets have been created that reflect the designed work.
- Tickets have a complete definition of done. Any comments in the Github tickets have been consolidated/summarized in the top level comment. In short, tickets should be able to be estimated by the team without scrolling through a giant set of comments.
- You have an idea of how to execute the work (i.e. where are the dependencies, what can be done in parallel, what will be demonstrated in the sprint demo). This is essentially the engineering plan to deliver the work. (PM will help model the plan in the Zenhub tickets prior to this meeting).
- You have verified that the issues that you created represent the full scope of work with PM.
<<<<<<< HEAD
<<<<<<< HEAD
- You have reached out to your manager the Monday before so that they can put up the issues on PokerBot for asynchronous estimation.

### During the Sprint

Your primary job is to ensure that the delivery continues to be unblocked.

This could involve:

=======
- You have reached out to the Scrum Master the Monday before so that they can put up the issues on PokerBot for asynchronous estimation. 
=======
- You have reached out to the Scrum Master the Monday before so that they can put up the issues on PokerBot for asynchronous estimation.
>>>>>>> Fix linting issues.

### During the Sprint

Your primary job is to ensure that the delivery continues to be unblocked.

This could involve:
<<<<<<< HEAD
>>>>>>> First draft of Epic Leads process.
=======

>>>>>>> Fix linting issues.
- Scheduling meetings with required folks (including PM) to resolve any unexpected product requirement questions.
- Creating temporary Slack channels to resolve open questions.
- Being the point of contact for any questions related to the work.
- Generally just keeping up to date on the progress and helping resolve blocks and/or escalating as needed.
- As parts of the feature land, ensure that they are working as expected and coordinate with PM to help shake out any bugs.
- Also, doing some of the work :) You will be responsible for coordinating the deployment (potentially via feature flags) and any work related to integrating frontend and backend work.

## Landing the Epic

Congratulations! Your Epic has landed in prod! There are few last steps to be completed to tie up loose ends.

### During Sprint Retrospective

Use the retrospective to look back at how the Epic went. What went smoothly and what didn't? The guidelines in this document are simply that. Guidelines. What would you add, what would you remove? Use the retrospective as an opportunity to lead conversations on how we can improve Software Engineering on our team whether that be related to our team processes or our engineering practices/quality.

### Before the Sprint Demo

Schedule a quick meeting with PM and UX to go through how you'd like to demo the new thing before the demo itself. Figure out talking points.

### Bugs

<<<<<<< HEAD
<<<<<<< HEAD
It is your responsibility to ensure that the feature lands to spec during the Implementation Phase. However, if there any bugs, please coordinate with PM to figure out when to address any gaps and resolve as quickly as possible. Coordinate during subsequent Drafting the Sprint Backlog meetings to ensure that high priority bugs get pulled into the following sprint.
=======
It is your responsibility to ensure that the feature lands to spec during the Implementation Phase. However, if there any bugs, please coordinate with PM to figure out when to address any gaps and resolve as quickly as possible. Coordinate during subsequent Drafting the Sprint Backlog meetings to ensure that high priority bugs get pulled into the following sprint.
>>>>>>> First draft of Epic Leads process.
=======
It is your responsibility to ensure that the feature lands to spec during the Implementation Phase. However, if there any bugs, please coordinate with PM to figure out when to address any gaps and resolve as quickly as possible. Coordinate during subsequent Drafting the Sprint Backlog meetings to ensure that high priority bugs get pulled into the following sprint.
>>>>>>> Fix linting issues.
