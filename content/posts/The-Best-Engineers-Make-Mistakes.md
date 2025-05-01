+++
title = "The Best Engineers Make Mistakes"
date = 2025-05-01T23:00:00.000Z
description = "The best engineers aren't perfect - they're resilient. Learn practical steps to handle mistakes, move from panic to problem-solving, and create a culture where engineering errors become valuable learning opportunities.RetryClaude can make mistakes. Please double-check responses."
keywords = [
  "",
  "Incident Response",
  "Learning from failures",
  "Blameless culture",
  "Engineering mistakes"
]
categories = [
  "",
  "productivity",
  "engineering",
  "personal development",
  "psychology"
]
+++

You've just pushed code that took down production. Your heart races as catastrophic scenarios flood your mind. When you realise you've made a significant mistake, this moment tests you like no other.

## The Reality of Engineering Mistakes

Engineering is a profession where we build complex systems with thousands of moving parts, often under time pressure, with incomplete information and questionable dependencies. The question isn't whether you'll make mistakes but how you'll respond.

What separates exceptional engineers isn't an absence of errors but how they handle these incidents and turn them into opportunities for growth. 

A mistake that leads to no change is your only actual failure.

## The Psychological Journey

The first challenge in addressing any engineering mistake is managing your mind. Your brain's immediate response to disaster is rarely at its best. The panic response, racing heart, tunnel vision, and catastrophic thinking work against clear thinking precisely when you need it most.

Recognise this state for what it is: a temporary condition triggered by your panic that does not reflect your worth as an engineer.

## From Panic to Progress

### 1. Acknowledge Reality Without Judgment

When things go wrong, start by accepting the situation with minimal self-criticism. Observe what's happening with indifference: "The production database is down" rather than "I've ruined everything."

### 2. Recovery Before Root Cause

Focus first on restoring service and then on understanding what happened. This means implementing the most direct path to recovery before diving into why the mistake occurred. Recovery might mean rolling back a deployment, failing over to a backup system, or implementing a temporary workaround. Whatever returns service to users fastest.

### 3. Extract the Learning

Once systems are stable, conduct a blameless postmortem. The goal isn't to assign fault but to understand contributing factors. What conditions made this mistake possible? What safeguards were missing? What assumptions proved incorrect? The answers to these questions are where the real value lies.

## Common Engineering Mistakes and Better Responses

### When Security Vulnerabilities Are Introduced

Scenario: A code review reveals a potential security issue in your recently deployed code.

Panic response: "We could be compromised already, and I'll be responsible for a breach."

Better response: "I've identified a security exposure in my recent changes. I've implemented immediate mitigation measures and am working with security to assess potential impact."

### When Data Integrity Is Compromised

Scenario: A database migration resulted in unexpected data transformation.

Panic response: "Critical data is corrupted, and our customers will lose trust."

Better response: "Our database operation has caused unexpected data transformation. I've stopped further changes and preserved the current state while preparing recovery options."

### When Infrastructure Changes Have Unexpected Consequences

Scenario: Services are failing after your recent infrastructure updates.

Panic response: "The entire system is down because of my infrastructure change."

Better response: "My infrastructure change has had unexpected consequences. I've rolled back."

\
The panic responses in these scenarios are perfect examples of "rumination loops" and "catastrophising" in a previous post on [Troubleshooting the Engineer's Brain](https://ianduffy.ie/2025/04/15/troubleshooting-the-engineers-brain/). These patterns can lock us into unproductive mental cycles that prevent effective problem-solving.

## Building a Blameless Engineering Culture

Individual responses to mistakes matter, but team culture determines whether lessons spread or remain isolated.

### Document and Share Learnings

Create a knowledge base of incident reports focusing on learning rather than individual blame. When engineers openly share their mistakes and lessons, the team benefits from each person's experience.

For smaller learnings, consider creating a "#til" (Today I Learned) Slack channel where team members can share small daily learnings, including minor mistakes and their solutions. This will normalise the learning process and create continuous knowledge sharing rather than waiting for major incidents to drive improvement.

### Reward Transparency Over Perfection

Explicitly recognise and reward engineers who identify and address their own mistakes. This reinforces that the goal isn't perfect performance but continuous improvement.

This approach helps counter the "perfectionism paralysis" discussed in [Troubleshooting the Engineer's Brain](https://ianduffy.ie/2025/04/15/troubleshooting-the-engineers-brain/), where engineers delay submitting work because it doesn't feel "ready" despite meeting all requirements.

### Schedule Regular System Improvement Time

Allocate dedicated time for addressing issues identified during incidents. Without this commitment, the pressure to deliver new features will override fixing the underlying issues.

## Reframe Mistakes into Learnings 

The most powerful reframing for mistakes is seeing them as learnings. 

Mistakes are inevitable in engineering. Growth from those mistakes is optional. Choose growth.
