---
title: "The industrialization of code"
layout: post
author: "Axel Vanraes"
categories:
  - "AI"
  - "SoftwareDevelopment"
  - "HealthTech"
---

![An assembly line that takes in source code and packs finished software boxes.](/assets/software-factory.png)

## 1. Opener — the failure (automating feedback almost burned us out)

- 2 months into building our EHR, after 5 years as a company
- Were already coding with AI, but human-in-the-loop, babysitting PRs
- This project: u-turn in how we approach development
- After 2 months: clear that software development has changed a lot, changed forever
- First approach: "let's automate all user feedback"
- Built in-app feedback tool → automatically translated feedback into GitHub issues
- Issues picked up by agents → turned into PRs → automated builds of the app
- Human could evaluate immediately
- Result: quickly saturated our PR backlog, created a bottleneck
- Cognitive overload of reviewing PRs was almost at a point that it could create burnout
- Realization: need to invest in primitives and patterns — benefit on multiple layers
- **Punchline candidate:** throughput of code generation is not the bottleneck anymore — review and consistency are

---

## 2. Human-in-the-loop vs human-out-of-the-loop

- Human-in-the-loop coding with AI versus human-out-of-the-loop: totally different approach
- Approach taken: automate user feedback and error reports into a testable version of the app, end-to-end, without a human in the loop as much as possible
- Obviously parts not automatable → automatic triage agent picks those apart
- All automated stuff executed fully today → can then be merged and shipped
- Result of triage: simple things can be automated, hard things stay hard
- "Simple" is a deliberate word here — simple is important, different from easy
- Hard things stay human-loop: difficult UX problems, patient identification, odd problems, patient merge, lifecycle events, complex UX problems like planning and scheduling
- Hard things stay hard because: need to be very much in touch with customers and the state of the art
- Too much strategy involved
- **Reframe to consider:** not "human out of the loop" but human moved to a different point in the loop — from writing/babysitting PRs to owning patterns, evals, and the hard strategic problems
- **Note for healthcare framing:** be explicit about where review/oversight sits, since this is an EHR — readers will ask

---

## 3. The industrialization of code

- Conclusion: writing code has become more industrialized
- If you can identify a simple problem and a standard pattern for the solution → almost a script an agent can follow to implement it
- Pattern is well documented
- Review process straightforward: review is basically asking an agent "did the feature follow the pattern or not?"
- Standardization and processes came down to: writing patterns, providing a component registry, other scaffold tooling
- Scaffold tooling makes sure standards are followed — because under the details, agents are always capable of doing something slightly different
- Industrialization of code: almost no room to implement features in a specific ad hoc way, no room for a semi-artistic view of how to approach a problem
- Risk: too many variations in approaching problems in the codebase
- Agents start to pick it up, start to introduce anti-patterns
- Code quality goes down quickly
- A consistent codebase, using the same pattern all over again, all over the place, reinforces agents to do the same thing in the future
- **Punchline candidate:** consistency compounds, variation teaches agents anti-patterns

---

## 4. The new gap — maintaining the machine that writes the code

- Introducing patterns as skills, enforcements as hooks, scaffold tools, etc.
- While creating a new layer of tooling for agents to work with, also need to maintain it — proper maintenance and test environment for these tools
- Need to do so becomes higher — this is the biggest gap we are experiencing right now
- Skills, hooks, tools all need to be tested
- Comes down to keeping a collection of prompts and evals
- Eval = benchmark set that allows you to test skills, prompts, hooks
- See if you change the prompt, results are still good
- Way forward to: test new models, see if they perform equally well; test other model providers; see how quickly you can swap out a model provider for another one
- **Note:** evals as benchmark set = model/provider portability, worth stating explicitly as a concrete benefit

---

## 5. Closing — not a free lunch

- Absolutely not a free lunch
- Really requires rethinking the whole development process, the whole approach to code
- Need to detach yourself, your value, your identity from the code
- Think more about how to set up the process so the code that comes out of it is good and reliable
- Once you get the human out of the loop, that's where the real value appears — frees up your mind to focus on other things

---

## Gaps to fill before full draft (from earlier discussion)

- [ ] Rough numbers: volume of feedback items, share auto-resolved vs. triaged to humans, PR review time before/after patterns
- [ ] One concrete worked example: a real feedback item that went feedback → issue → agent PR → shipped, no human writing code
- [ ] Decide explicit framing on human oversight/review given EHR = healthcare software
