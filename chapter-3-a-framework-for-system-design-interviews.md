# CHAPTER 3: A FRAMEWORK FOR SYSTEM DESIGN INTERVIEWS

## A 4-step process for effective system design interview

### Step 1 - Understand the problem and establish design scope (3-10 minutes)

* What specific features are we going to build? • How many users does the product have?
* How fast does the company anticipate to scale up? What are the anticipated scales in 3 months, 6 months, and a year?
* What is the company’s technology stack? What existing services you might leverage to simplify the design?
* .......

### Step 2 - Propose high-level design and get buy-in (10-15 minutes)

* Come up with an initial blueprint for the design. Ask for feedback. Treat your interviewer as a teammate and work together. Many good interviewers love to talk and get involved.
* Draw box diagrams with key components on the whiteboard or paper. This might include clients (mobile/web), APIs, web servers, data stores, cache, CDN, message queue, etc.
* Do back-of-the-envelope calculations to evaluate if your blueprint fits the scale constraints. Think out loud. Communicate with your interviewer if back-of-the-envelope is necessary before diving into it.

If possible, go through a few concrete use cases. This will help you frame the high-level design. It is also likely that the use cases would help you discover edge cases you have not yet considered.

Should we include API endpoints and database schema here? This depends on the problem.

### Step 3 - Design deep dive (10-25 minutes)

At this step, you and your interviewer should have already achieved the following objectives:&#x20;

* Agreed on the overall goals and feature scope
* Sketched out a high-level blueprint for the overall design
* Obtained feedback from your interviewer on the high-level design
* Had some initial ideas about areas to focus on in deep dive based on her feedback

You shall work with the interviewer to identify and prioritize components in the architecture.

### Step 4 - Wrap up (3-5 mintues)

In this final step, the interviewer might ask you a few follow-up questions or give you the freedom to discuss other additional points.

* Bottlenecks and discuss potential improvements.
* Refreshing your interviewer’s memory by recapping your design.
* Error cases (server failure, network loss, etc.)
* Operation issues are worth mentioning. (monitor metrics, error logs, roll out the system)
* How to handle the next scale curve.
* Propose other refinements.

## Dos

* Always ask for clarification. Do not assume your assumption is correct.
* Understand the requirements of the problem.
* There is neither the right answer nor the best answer. A solution designed to solve the problems of a young startup is different from that of an established company with millions of users. Make sure you understand the requirements.
* Let the interviewer know what you are thinking. Communicate with your interview.
* Suggest multiple approaches if possible.
* Once you agree with your interviewer on the blueprint, go into details on each component. Design the most critical components first.
* Bounce ideas off the interviewer. A good interviewer works with you as a teammate.&#x20;
* Never give up.

## Don'ts&#x20;

* Don't be unprepared for typical interview questions.
* Don’t jump into a solution without clarifying the requirements and assumptions.
* Don’t go into too much detail on a single component in the beginning. Give the high- level design first then drills down.
* If you get stuck, don't hesitate to ask for hints. • Again, communicate. Don't think in silence.
* Don’t think your interview is done once you give the design. You are not done until your interviewer says you are done. Ask for feedback early and often.
