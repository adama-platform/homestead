# Homestead
A personal ecosystem for AI slop with SlopBox orchestration
------

## Introduction

We live in the era of AI, and this project continues goober (https://github.com/mathgladiator/goober). At its core, I want to radically simplify how I think about the boundary between infrastructure and product. AI is both amazing and shockingly not that great, often in the same afternoon. Goober showed what happens when you give a chatbot the ability to control and ship working product code live, and that part was a success. It also started to buckle under its own weight. Every good idea landed in the same host and the same manifest, and eventually the thing meant to contain the chaos became the source of it.

Homestead takes the parts of goober that worked and refactors them. The goal is no longer one clever host. The goal is layers, each narrow enough that an agent can own it and ignorant enough of everything above it that it can't wander off. I want to "ship org" with agents: build the kind of structure a well-run company uses, where each team has a clear job, a clear interface, and no reason to call a meeting.

## What goober taught me

**An agent must be focused.** An agent does its best work when the job looks like a fixed-function pipeline: this goes in, that comes out, here is how you know you were right. Give it an open field and it gets confused, then confident, then wrong. So every piece of homestead has one job. Pond holds data. Fork checks SQL. Barn runs boxes. Rancher holds the routes. Sun keeps time. Permit knows who you are. None of them needs to understand the others to do its job well, and an agent working on one of them only needs that one in its head.

**Avoid bureaucracy.** Agents love to chat amongst themselves to reach consensus. It looks like collaboration, but it is a kind of mode collapse, and it burns tokens like a committee burns afternoons. You can't fix this by asking them to talk less. You fix it by giving them less to talk about. When the surface area of what an agent can do is small and well-defined, there is nothing to negotiate. This is focus again, seen from the other side: a narrow job doesn't need a meeting.

**Move fast with bulk approvals.** My big mistake in goober was assuming a human has the stamina to keep up with every change. I don't, and neither does anyone else. Agents don't get tired, and a human approving one change at a time becomes the slowest part of the machine and then the most careless part. Instead, I want a beta version of the product that moves as fast as the agents can go, and a production version that moves when I say so. Between them is one large diff, and I review it the way an executive reviews a quarter: by its shape, its risks, and what broke along the way, not by reading every line.

**Manifests were spiritually right, but cumbersome.** The manifest said everything a box would ever touch, and that was the right instinct. In practice it grew a section for every idea, every section had to agree with every other, and changing anything meant re-deriving the world. Homestead breaks it apart. A box just exists, identified by its hash, and declares only the buckets it may ever write. Routes carry everything else: who may call, which objects a request can touch, which tables it can read. Some of that is fixed when the box is deployed, and some of it is decided per request at runtime, and the split between those two is where the access control gets its flexibility.

**Organizational limits apply to AI too.** We invented teams, interfaces, and job descriptions because a human can only hold so much in their head at once. Agents have the same problem with a different name: context. Every extra thing in the context window costs tokens and dilutes attention. The structures that keep a company sane, small teams with clear ownership and narrow contracts between them, keep agents sane for the same reasons. I don't think this is a coincidence.

## Quality is the bottleneck

With all of that in mind, I'm thinking more holistically about quality, because quality is the thing holding agentic work back. Not capability. The models can write the code. What they can't do yet is know when the code is wrong, and what I can't do is read all of it to find out.

Sure, we could just trust the AI and let it go, and there is a case to be made for that when the stakes are low. But the interesting work isn't low stakes. So the question homestead asks is: how small can you make the space where a mistake can happen, and how quickly can you see one when it does?

The answer I'm betting on has three parts. First, constrain what a box can do so the worst mistakes are impossible, not merely discouraged. A box can merge into the objects its route resolved, and nothing else. Second, make failures loud and legible to the thing that will fix them. When something breaks at runtime, it becomes a problem: a structured record an agent can read, with what broke, where, how often, and a hint. Logs are for people. Problems are for the agent writing the next version. Third, keep the human at the boundary, where judgment actually works. I can feel whether a route should be able to touch a user's profile. I can't feel whether line 4,000 of generated code is right.

## Where I stand

The human in this picture is an owner, not a reviewer. I set the boundaries, approve the big diff from beta to production, watch the problem log, and decide what the agents work on next. The agents do the thousand features. The system makes sure that whatever they get wrong stays inside a box.

That is why this is goober reconsidered as homestead. Goober was one big house. Homestead is a piece of land with a few buildings on it, each with a job, a door, and a fence.
