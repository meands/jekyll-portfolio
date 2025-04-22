---
title: Schema-Driven Form Design (Revisited)
date: 2025-04-20 00:00:00 +0000
tags: [forms, schema]     
description: Examining the evolution of a schema-driven form system, focusing on challenges with dynamic schema modifications, traceability, and flexibility. It highlights key lessons learned and the importance of clear context-based adjustments in complex form systems.
---

## Looking Back

About a year ago, I introduced a schema-driven pattern for building complex forms into a data registration platform. It particularly helps on forms with nested conditional fields, repeated UI logic, and shifting requirements. The core idea was simple: define a flat schema where each entry declares:

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

### Mount v.s. Visibility

One unexpected issue was the distinction between mounting and visibility. `renderWhen` was originally the catch-all for conditional logic — it controlled whether a field was rendered at all.

But this became problematic when we needed to temporarily hide fields without unmounting them — to retain their values. One use case involved a checkbox that toggled advanced settings. Using renderWhen here reset the fields on unmount, breaking the experience.

To solve this, we introduced an additional internal state holder to track UI-specific metadata like hidden, disabled, or prefilled states — effectively overriding the schema’s static declarations. This allowed us to separate concerns between mounting and visibility.

### Imperative User Flows

Some behaviours are inherently event-driven rather than state-driven. There is a difference between *the value of something is x*, and *the user selecting the value to be x*. 

For example, after a lookup, you want to disable a field, fetch new values, or inject an entirely new set of fields. However, this cannot be expressed as *disable/fetch/inject when <...>*. These kinds of action are better suited to an imperative layer on top of the declarative schema, interacting with the same internal state holder used for visibility/disablement. This provided a controlled escape hatch for workflows that couldn’t be expressed declaratively.

### The Schema Isn’t Always Enough

Originally, the idea was that the schema should fully define every possible form permutation — a single source of truth for structure, logic, and validation.

In practice, this proved too rigid. We often needed to render the same form differently based on external context: user role, feature flags, or preloaded data. Sometimes this meant pre-filling fields. Other times, it meant hiding or changing the behavior of certain sections.

To accommodate for contextual adjustments, we introduced a **schema modifier** — a function that take a base schema and returns a slightly adjusted version. The modifier executes at runtime, tailoring the schema before rendering begins:

```ts
// modifier.ts
function modifySchema(schema, context) {
  if (context.user.role === 'readonly') {
    // Imperatively walk and mutate the schema
    schema.sections = schema.sections.filter(s => s.id !== 'advanced');
    schema.fields['comments'].disabled = true;
  }

  if (context.featureFlags.useNewLayout) {
    schema.layout = 'grid';
  }

  return schema;
}
```

This solved the immediate problem and allowed us to reuse the same base schema across different use cases. However, by modifying the schema dynamically, we lost some of the predictability and clarity we initially had. It became harder for developers to trace what the final shape of the form would be in any given context. Consider:

- Which fields will render, and why
- Whether a field is being hidden, disabled, or removed entirely
- Which parts of the schema are static, and which are context-dependent

Understanding the form now required knowledge of not just the base schema, but also the transformation logic and the inputs that drove it.

This eroded one of the original strengths of the schema-driven approach: centralisation and transparency. Instead of looking at one place to understand how a form behaves, developers now had to track context, modifiers, and state — reintroducing some of the cognitive load we were originally trying to avoid.

The system was evolved into **schema transformations**.

We now define a base schema, and a set of contextual transformations — small, composable functions with narrow responsibilities. Each transformation applies a well-scoped change:

```ts
// transformations.ts
export const hideAdvancedFields = schema =>
  removeSection(schema, 'advanced');

export const disableComments = schema =>
  setField(schema, 'comments', { disabled: true });

export const applyGridLayout = schema =>
  setLayout(schema, 'grid');

// usage
let schema = baseSchema;

if (context.user.role === 'readonly') {
  schema = pipe(schema, hideAdvancedFields, disableComments);
}

if (context.featureFlags.useNewLayout) {
  schema = applyGridLayout(schema);
}
```

Each transformation is:

- Isolated and reusable
- Context-aware
- Easy to trace and reason about 

### UX Based on Usage

Another challenge we've faced recently is knowing when a form has diverged enough in its behavior or structure to justify creating a separate schema altogether.

Take this example: you’re maintaining a volunteering platform where users can register to volunteer across different causes—soup kitchens, charity shops, animal shelters, etc. Early on, a single, unified schema might be sufficient to handle all these cases, especially if the differences between them are minimal.

But over time, usage data reveals that the majority of users are registering for dog shelters. Naturally, product and UX teams start tailoring the experience: pre-populating certain fields, removing irrelevant ones, and adding extra context or options specific to dog shelters.

These changes often happen incrementally. Initially, the base schema simply adds a renderWhen condition or tweaks a label. But as more dog shelter-specific requirements accumulate, the schema becomes increasingly specialised, with only a handful of truly generic fields remaining.

At this point, the form looks like a generic one, but behaves like a domain-specific flow. Everything is still technically "held together" in a single schema, but the logic becomes harder to reason about, and maintenance becomes error-prone.

## Final Thoughts

The schema-driven pattern still holds up. It separates business logic from UI, simplifies conditional rendering, and makes forms easier to reason about — most of the time.

But as the system grows, flexibility cuts both ways. The biggest challenge we now face isn’t what the system can do, but how easily developers can understand it. When schemas are dynamic and context-aware, traceability becomes critical. 

Our focus has shifted from expressiveness to clarity: making it easy to see what’s happening, why, and where.

When designing a form system with complex branching logic or contextual behaviour, schema-first is still a strong starting point. Just keep in mind the question: “What shape is the schema right now, and why?”.


