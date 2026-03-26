---
title: "Pull The Plug Modeling: A reset before you model"
date: 2026-03-25
description: "PTPM is a mindset that asks you to imagine your domain without electricity. Just people, paper, and pens. It removes technical noise so the real domain becomes visible."
tags: ['Domain Modeling', 'Software Architecture']
---

I work as a software architect for an Indigenous organization. Twice a year, we get together offsite for a team retreat. We reflect on our values, challenge our assumptions, and reconnect as a team. We often invite an Elder to open the event with a prayer.

I am not a spiritual person. But I have always been struck by what that prayer accomplishes. Everyone arrives carrying something different. Different concerns, different frames of reference, different languages. The prayer does not erase those differences. It suspends them, just long enough for everyone to find common ground before the work begins.

It is a reset. A deliberate, collective reset. A way to remove our blinders and truly listen to one another.

That kind of reset is rare in most software and modeling projects I have worked on. Everyone comes with their own assumptions. And the work suffers for it.

## Technology blinds us

Technology is not neutral. It shapes the way we think about problems before we even start solving them.

Developers turn to familiar patterns. Validation rules, normalized tables, unique constraints. Not because the problem requires it. Because the tools are right there.

Clients try to speak technical. They ask for dropdowns and mandatory fields. They think that is what developers need to hear.

Everyone around the table is taking shortcuts. Shortcuts that feel productive but pull the conversation away from what actually matters: understanding the domain.

We do not need better tools. We need to step back from the tools entirely. We need a deliberate, collective reset.

## Pull the plug

When a discussion starts drifting toward technical details too early, I pull the plug. I ask people to imagine how the work would get done in a world without electricity, software, or screens. Just people, paper, and pens.

This is what I call Pull The Plug Modeling. It is not a methodology. It is not a framework. It is a mindset. A deliberate decision to imagine the domain as it exists in the real world, before any technology existed to support it.

Think of it as the opening prayer of a modeling session. It realigns the room.

When you remove electricity from the conversation, something shifts. The technical shortcuts disappear. Nobody talks about dropdowns or API endpoints. People start describing what actually happens. Who does what. What information travels from one person to another. What gets written down, and why.

The domain becomes visible.

## What a world without electricity reveals

In PTPM, we imagine ***humans*** exchanging ***information*** on ***paper***. Each of these three components defines an axis of analysis: the physical constraints of the medium, the human factors at play, and the characteristics of the information exchanged. The goal is not to tell us what decisions to make, but to force us to ask better questions and think differently about our domain.

### Physical constraints

What the materiality of the paper world imposes as concrete limits, and what those limits reveal about the true nature of exchanges.

**Everything is slow**

Information travels at the speed of a walking human. When a message takes time to arrive, every step of the journey becomes visible. You see who did what, in what order, and why. Slowness is not a flaw. It is a revealer.

It also exposes the asynchronous nature of work. In a paper world, nobody waits for an immediate response. You send, you continue, you wait. Synchronous communication is a costly exception reserved for situations that truly require it. When you pull the plug, you realize that most exchanges do not need to happen right now.

**Everything is tangible**

A sheet of paper exists in one place at a time. A document cannot be modified by two people simultaneously. It belongs to someone. That possession is not a formality. It is a responsibility. Until a document is handed off, its author is accountable for it.

Handoff is an explicit act. You pass the document, you transfer the responsibility. Retrieving a document already in circulation is also an explicit act. Nothing changes hands without someone deciding it consciously.

**Mistakes are permanent**

Written in pen, you cross out, correct, rewrite, but you do not erase. Every correction leaves a trace. You can see that something changed, and what it was. The medium makes corrections visible. In a digital world, you can modify without leaving a mark. In a paper world, every correction is visible and traceable.

**Deletion is physically hard**

Destroying a document takes deliberate effort. It is more natural to add than to erase. When a piece of information is no longer relevant, nobody goes back into the archives to apply liquid paper over its name in every file. You pull it from circulation. You add a new document that says reality has changed. The archives retain their value, even for information that has become obsolete. Deletion does not really exist in a paper world. It is a technological abstraction that pulls us away from what is actually happening in the domain.

### Human factors

What the absence of automation puts back in human hands, and what that reveals about the central role of judgment in any system.

**Nothing happens on its own, everything is intentional**

In a paper world, nothing happens without someone deciding to make it happen. Every step in a process is an explicit human decision. Business rules are visible because someone has to apply them manually.

The vocabulary of the domain describes what is actually happening. Nobody says "delete." They say archive, approve, forward, cancel. Actions have business names that encode the intent, not the mechanics.

**Validity is a human decision**

It is the person receiving the document who judges whether the information is sufficient to act on. In a paper world, an incomplete form is not invalid. It is in progress. The person who receives it knows what to do with what is missing. They follow up, they proceed, they escalate. A process is not blocked because a field is empty.

Judgment is contextual and practical. A partial address can be enough if you can locate the place. It is the human who decides, not the system.

### Information characteristics

What a paper document requires to be useful, and what those requirements reveal about how information should be structured.

**A document is self-contained**

A sheet of paper must be understandable on its own. The necessary context is included directly in the document. A document should carry enough information to be understood without looking elsewhere. External references are a conscious friction, not a reflex.

**Duplication is normal**

Repetition is a natural consequence of autonomy. The same name appears on ten different forms. That is not a problem to fix. It is a response to a real constraint. You do not normalize on principle. You normalize when duplication causes a concrete problem. Eliminating duplication has a cost you accept only when the benefit is real.

## When software replaces the domain

PTPM is not only useful when designing something new. It is perhaps most valuable when replacing something that already exists.

When a software system has been in place for years, something subtle happens. People stop describing their work. They start describing the software. They talk about screens, menus, and buttons. They ask for the same interface with one more field. The original domain fades behind the system that was built to support it.

Worse, they adapt to the software's limitations without noticing. They invent workarounds that become habits. They forget that certain friction points come from a bad design decision made years ago, not from the nature of their work.

When you pull the plug in that context, you are helping people rediscover what their work actually looks like, not as the software has trained them to see it.

A bad software system does not just create technical problems. It gradually reshapes the perception people have of their own domain. PTPM is a way to undo that.

## A reset in harmony

The opening prayer at our team retreats does not solve anything on its own. It does not resolve the tensions in the room. But it creates the conditions for a better conversation. Everyone arrives at the same starting point, with the same openness, before the work begins. PTPM does the same thing.

It does not give you answers. It clears the space where the right questions can finally be asked.

Pull the plug. See what remains. Start from there.