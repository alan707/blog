---
title: "Death by a Thousand Questions: Working With Problem Admirers"
date: 2026-05-10
tags: ["engineering-culture", "design-reviews", "communication", "career"]
draft: false
---

> A *problem admirer* is someone who falls in love with everything that could go wrong with your idea, and forgets that the idea was supposed to fix something.

**TL;DR.** Problem admirers feel like blockers, but if you handle them right *before* the design review, they're often the most valuable reviewers you have. Their objections become your FAQ. Ignore them and you ship with avoidable wounds. Listen too much and the project dies in committee. This post is about threading that needle.

<figure style="text-align: center;">
  <img src="bertram_gilfoyle_in_silicon_valley.webp" alt="Bertram Gilfoyle from HBO's Silicon Valley" style="display: block; margin: 0 auto; max-width: 100%; height: auto;">
  <figcaption>Every org has its Gilfoyle. <em>Silicon Valley</em> (HBO).</figcaption>
</figure>

### The 400-Repo Tax

A friend was telling me about a company he works at with around 400 repos. Someone proposed a small process change: let three trusted staff SREs bump a shared internal API version across all repos without per-team approval. The change was mechanical and didn't need per-repo review.

The proposal didn't get rejected. It died by a thousand questions.

```
Are we sure we want to override the existing process here?

What problem are we actually trying to solve? Things seem to be working.

Why are we taking on a risky new operation when nobody's asked for it?

Shouldn't every repo owner have a say in how their repo gets changed?
The day this breaks something, they're going to come straight at us.
```

Each question was reasonable in isolation, but cumulatively they were fatal. The proposal sat. Months passed. Eventually 400 PRs went out the normal way, the way the proposal was designed to avoid. Some never landed. The version drift the proposal was meant to *prevent* got worse the longer the proposal was being debated.

If you've worked at any company larger than about 15 engineers, you already know this person. Every org has one. They're often senior, often respected, often technically excellent, which is exactly why the room takes their objections seriously. They're not the loud cynic everyone has learned to discount. They're the careful, thoughtful person whose nine "but what about" Google Docs comments will get treated as nine real questions, even when the cumulative effect is paralysis.

*(If you can't think of who yours is, there's a non-zero chance it's you. We'll come back to that.)*

I'll call these folks **problem admirers**: people who genuinely love thinking about everything that could go wrong with your idea, and who, with the best of intentions, will talk a good proposal into a slow death.

Most of them aren't blockers. They're risk-sensitive engineers using the wrong interface.

### The Other Failure Mode

The 400-repo story is what happens when problem admirers win. There's a symmetric failure mode I want to be honest about, because I've lived the other side of it.

A few years back I proposed using Docker for our hardware-in-the-loop (HIL) orchestration computers. The premise was clean. Every automated test should run against an identical set of dependencies. No more *"works on my HIL rig."* Pin the Dockerfile, ship the image, run the same thing everywhere.

It was a good idea. It still is.

There was a problem admirer in the org at the time. They had concerns. Specific ones. Something about local dev getting harder. Something about hiding context the engineers used during debugging. Something about the inner loop on a working rig.

I steamrolled. I had leadership's blessing and expectations to deliver. The objections felt like a personal nitpick, not real engineering arguments. So I shipped it org-wide.

The complaints came in waves over the next quarter. Local dev *did* get harder. The inner loop *did* slow down for engineers who used to iterate against bare metal. The Docker layer *wasn't* super ergonomic. None of these killed the project. But they were real, they cost engineering time, and they were almost exactly what the problem admirer had flagged in the design doc comments. The ones I'd waved off with *"will address in subsequent revisions"* or something equally hand-wavy.

The version of that project that listened to them would have been the same project, six weeks later, with three small mitigations baked in. I would have spent those six weeks doing something else. Instead I spent them responding to Slack pings about real blockers: HIL benches that wouldn't run, CI red from automated tests failing.

So: two failure modes on the table.

- **Over-listen.** The 400-repo case. Good ideas die in committee.
- **Under-listen.** The HIL case. Good ideas ship with avoidable wounds.

The strategies in this post are about threading that needle.

### Why We Still Need Problem Admirers

It took me a while to internalize this, but the problem admirer is, more often than not, *helping*. The trick is that the help arrives in a costume that looks like obstruction.

A few reframes that helped me get there.

**They make the idea better.** Every "but what about X?" is a chance to find the sharp edges early, before the review turns into discovery. The objection that derails a design review is often the same issue you would have discovered in production six weeks later. Better to spend an hour on it now.

**They surface institutional memory.** They remember the 2021 attempt that failed, and *why*. They know why that library still needs the local patch, why nobody dares bump that dependency, which "simple" migration is anything but. New hires can't give you that. Even staff engineers two years in usually can't. The problem admirer often can, and the proposal you write after talking to them is one that doesn't repeat a mistake your org has already paid for.

**Their objections become your "Alternatives Considered" section.** Which, if you've ever shepherded a design doc through a real review, is the section that gets the doc approved. Reviewers want to see that you've thought about the obvious failure modes. Problem admirers will hand you that list for free, in writing, in the comments.

A reframe table I keep in my head when I feel my shoulders tightening at a comment thread:

| What it sounds like | What it actually is |
| --- | --- |
| "I don't think our team should merge into the monorepo, it'll slow us down" | The actual blocker for the migration, surfaced for free |
| "We tried this in 2022" | Free historical context |
| "Hope is not a strategy. Do we have any data on whether this is net faster?" | A reminder to put measurements in the design |

### Why The Dynamic Is So Hard

If problem admirers are net positive, why does working with them feel so terrible?

- **Asymmetry of effort.** A thirty-second objection generates three hours of work. *"What I'd like to see is validation that this won't break compatibility with a release that's five years old."* Twelve seconds to type, a week to run down. Multiply by nine comments and you've generated someone's quarter from an afternoon of doc-scrolling and morning coffee.
- **They rarely propose solutions.**  (I was going to leave this one blank because that is exactly what a problem admirer's comment would look like). Jokes aside, the lack of solutions seems a bit adversarial, but they feel like they're helping by pointing out the gaps. 
- **Public-channel objections feel like attacks**, even when they're not intended that way. When someone realizes their design doc is littered with comments, the proposer feels called out publicly.

And on their side, the part that's easy to forget: **they've often been right before**. They've watched proposals ship over their objections, hi, and seen the predicted issues land in production six weeks later. Their nitpicking is a *learned response* to organizations that didn't take risk seriously the last time around. The behavior that looks obstructive from your side often started as the right call in a previous role, and got calibrated up over a decade of being half-listened-to.

That doesn't mean you have to absorb every objection. But it does mean the dynamic isn't really about you. You're inheriting a relationship the problem admirer has with every proposal they've ever reviewed.

### The Fix: Work With Them Before the Meeting

If there's one sentence to take from this post, it's this: *never let a problem admirer hear your idea for the first time in the design review.* Everything else flows from that.

A handful of tactics that have worked for me, in roughly the order I deploy them.

- **Book a 20-minute pre-meeting 1:1.** *"I want to get your read on this before I send it around."* That's the whole pitch. Buys their best objections in a low-stakes setting and gives them ownership of the doc. The move I should have made before HIL.
- **Send a Slack message with the main points first.** Ask for feedback on each major point of your proposal. You'd be surprised how many things they might already agree on. Once you've identified the contention areas, use the meeting to highlight those.

> Not every objection deserves equal weight. A good objection changes the design, rollout, measurement, or rollback. A weak objection only expands the possibility space. Your job isn't to answer every possible concern. It's to separate material risk from speculative drag.

### In the Meeting Itself

The meeting shouldn't be a rubber stamp just to get the design through. You genuinely want feedback on rollout strategy, timeline, and the contention areas surfaced earlier.

- **Open by thanking your problem admirer by name** for their pre-review feedback. Signals to the room that the work has been vetted, and locks them into a collaborative posture publicly.
- **Time-box live, escalate async.** When a new objection lands, default to *"Great point, let's take that to the doc so we don't lose it."* Validates the question without surrendering the agenda. The doc is a much better venue for working through a real objection than fifteen people watching a clock.
- **Distinguish 'agreed' vs 'needs discussion' points** out loud. A live Google Doc with two columns helps: capture agreements, bubble up the ones that need more discussion. Don't try to rush this.

### A Note If You Are The Problem Admirer

I've been on the other side of this too. It's a useful role, and the org genuinely needs you. But there's a useful version and a destructive version, and the line between them is thinner than it feels from the inside.

**Assume good intent.** Start from the premise that the person proposing the idea is trying to solve a real problem, not sneak something bad past the org. They may be missing context, underestimating a risk, or aiming at the wrong layer of the system. That does not make the proposal careless. If you see a better path, help them get there. The useful version of this role is not pointing at the flaw and walking away. It is helping the team arrive at a solution that survives the concern.

**Propose a path forward, not just a no.** *"This won't work because X"* is one comment. *"What if we add eBPF observability to the existing pipeline first, and let the data tell us where the bottleneck actually is?"* is also one comment. The first ends the conversation. The second moves it onto constructive, additive ground.

**Choose the right channel.** If something is seriously wrong, reach out directly first. A quick Slack message lets the author understand the concern before the public review becomes a surprise debate.

**Notice when you're admiring instead of solving.** If you find the *problem* more interesting than any possible *solution*, that's a signal. The role you want to play is "rigorous co-author." The role you want to avoid is "person who enjoys watching ideas die." The first is rare and valuable. The second is common and corrosive, and it is harder to distinguish from the inside than most people think.

### Wrapping Up

The goal isn't to neutralize problem admirers. It's to channel them. The version of your design that survives a real problem admirer is a much better design. The version that *ignores* them ships with the wounds they predicted.

Back to where we started. The version drift kept getting worse while the proposal was being debated. That's the real cost of letting problem admirers run the room. Not the bad ideas they kill, but the good ones they delay into irrelevance.

So next time you're drafting a proposal, pick the person you're dreading the most, and book the 1:1 first. That one move usually changes the whole arc of the project.
