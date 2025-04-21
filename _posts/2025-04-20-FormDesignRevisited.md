---
title: Schema-Driven Form Design (Revisited)
date: 2025-04-20 00:00:00 +0000
tags: [forms, schema]     
description: Looking at the evolution of a schema-driven approach to manage complex forms. It explores how the system has adapted to new requirements—like contextual rendering, imperatively hiding fields, and handling dynamic UI states — while highlighting the challenges of maintaining clarity and traceability in a flexible system.
---

## Looking Back

About a year ago, I introduced a schema-driven pattern for building complex forms — particularly ones with nested conditional fields, repeated UI logic, and shifting requirements. The core idea was simple: define a flat schema where each entry declares:

- **What** it is (input type, label, etc.)
- **When** it should render (`renderWhen`)
- **How** it should behave (validation, info, disabled states)

The behavior of each form field — such as whether it should be rendered, required, or disabled — was defined declaratively using functions. These functions aren't just static rules; they are evaluated at runtime, meaning they have access to the current state of the form. As a result, fields could dynamically respond to user input or changes elsewhere in the form. For example, a field could be set to only appear if a user selected a specific option in a previous dropdown, and this logic would be cleanly captured within the schema itself.

This approach aimed to separate business logic from rendering logic. Now that it’s been in production for a year — used in multiple forms across a data cataloging platform — it feels like a good time to reflect on how it’s held up.

## Is It Working?

To varying degrees — yes.

The schema-driven form system is still powering every major form in the application. New features continue to use it, especially since conditional visibility is the norm in this domain (e.g., _"show these fields when 'Storage Type' is 'S3'"_). The pattern handles these use cases well, and developers working with the schema have mostly found it straightforward to work with.

That said, real-world complexity always pushes the boundaries of clean abstractions. Here are a few ways the original pattern has been challenged, and evolved.

## Lessons and Iterations

### Supporting Non-Field Displays

We began to receive requests for UI elements that weren’t directly tied to a form field — headers, info panels etc.

Rather than introducing a parallel rendering system, we kept it within the schema. These elements were added as schema entries with custom renderers. Since they could use `renderWhen` like any other field, they remained reactive and consistent with the rest of the system.

### The Schema Isn’t Always Enough

Originally, the idea was that the schema would be the single source of truth for the form. It would fully describe every possible state - what should appear, when they should appear, and how theyrepsond to the current form state. Under this assumption, every permutation of the form should be been expressible from this single object.

In practice, this assumption proved too rigid.

Requirements began to emerge that the schema alone couldn't express cleanly. We were asked to render the **same form structure** differently depending on external context, think a user's role, pre-existing data, and feature flags. 

Sometimes fields needed to be pre-filled. Other times, whole sections needed to be hidden. Occasionally, the same field needed to behave differently depending on the workflow it was part of.

To accommodate for this, we introduced a **schema modifier** — function that take a base schema and returns a contextually adjusted version. The modifier executes at runtime, tailoring the schema before rendering begins.

The flow shifted to:

```
    figure out context -> modify schema -> render form
```

This solved the immediate problem and allowed us to reuse the same base schema across different use cases. However, by modifying the schema dynamically, we lost some of the predictability and clarity we initially had. It became harder for developers to trace what the final shape of the form would be in any given context. Understanding the form now required knowledge of not just the base schema, but also the transformation logic and the inputs that drove it.

This eroded one of the original strengths of the schema-driven approach: centralisation and transparency. Instead of looking at one place to understand how a form behaves, developers now had to track context, modifiers, and state — reintroducing some of the cognitive load we were originally trying to avoid.

### 3. `renderWhen` vs `hideWhen`

One unexpected issue was the distinction between mounting and visibility. `renderWhen` was originally the catch-all for conditional logic — it controlled whether a field was rendered at all.

But sometimes, we *wanted* the field to remain in the DOM (preserving its value), while just hiding it from the user.

This came up when implementing a UX toggle: a checkbox that, when checked, would reveal some pre-populated advanced fields. Using `renderWhen` caused those fields to reset on unmount. 

