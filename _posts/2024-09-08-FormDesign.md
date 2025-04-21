---
title: Schema-driven Form Desgin
date: 2024-09-08 12:00:00 +0000
tags: [forms, schema]     
description: Schema-driven forms
---

Building dynamic forms is a commmon but often painful exprience for developers. As forms grow in complexity - with validations, error states, conditional logic, and ever-changing requirements — the code can quickly become cluttered and difficult to maintain. In this post, I’ll share a schema-based approach to form building that separates business logic from rendering, simplifies conditional rendering, and improves maintainability.

## The Problem with Traditional Form Development

There are a few recurring pain points when developing forms in modern applications:

- Visual Noise: Validation logic, error messages, and helper info often overwhelm the core rendering logic. It becomes difficult to see what’s actually being rendered.

- Changing Requirements: When requirements change (as they always do), developers must sift through deeply nested code across multiple components, making updates tedious and error-prone.

- Conditional Fields: The classic trap—nesting logic for conditionally rendered fields. When one field depends on the value of another, things get convoluted quickly.

Even with modern form state management libraries that handle field registration, validation, and touched states, handling dynamic field visibility or conditions usually ends up being imperative and messy. State management may be simplified, however business logic is an orthogonal issue.

## Existing Solutions

Schema-based form libraries like react-jsonschema-form attempt to solve this by defining forms declaratively. These libraries generate form UIs from a schema and reduce a lot of boilerplate code. 

React-jsonschema-form is a mature library that covers the grounds, however I do think there are issues when it comes to conditional fiels specifically. Nested conditions can be achieved via *deependencies*, consider an example with choosing the right vehicle:

```json
{
  "title": "Vehicle Details",
  "type": "object",
  "properties": {
    "vehicleType": {
      "type": "string",
      "enum": ["Car", "Bike"],
      "title": "Select a vehicle"
    }
  },
  "dependencies": {
    "vehicleType": {
      "oneOf": [
        {
          "properties": {
            "vehicleType": { "const": "Car" },
            "fuelType": {
              "type": "string",
              "enum": ["Electric", "Petrol"],
              "title": "Fuel Type"
            }
          },
          "required": ["fuelType"],
          "dependencies": {
            "fuelType": {
              "oneOf": [
                {
                  "properties": {
                    "vehicleType": { "const": "Car" },
                    "fuelType": { "const": "Electric" },
                    "batteryCapacity": {
                      "type": "number",
                      "title": "Battery Capacity (kWh)"
                    }
                  },
                  "required": ["batteryCapacity"]
                },
                {
                  "properties": {
                    "vehicleType": { "const": "Car" },
                    "fuelType": { "const": "Petrol" },
                    "engineSize": {
                      "type": "number",
                      "title": "Engine Size (litres)"
                    }
                  },
                  "required": ["engineSize"]
                }
              ]
            }
          }
        },
        {
          "properties": {
            "vehicleType": { "const": "Bike" },
            "motorized": {
              "type": "boolean",
              "title": "Is it motorized?"
            }
          },
          "required": ["motorized"],
          "dependencies": {
            "motorized": {
              "oneOf": [
                {
                  "properties": {
                    "vehicleType": { "const": "Bike" },
                    "motorized": { "const": true },
                    "motorPower": {
                      "type": "number",
                      "title": "Motor Power (W)"
                    }
                  },
                  "required": ["motorPower"]
                },
                {
                  "properties": {
                    "vehicleType": { "const": "Bike" },
                    "motorized": { "const": false }
                  }
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```

This definitely works and is strict and safe, and while the business logic is separated into a schema and sepearte from the rendering process, the readability is lacking.


## A Schema-Based Approach to Dynamic Forms

To tackle these issues, I built a form system inspired by schema-driven design but enhanced for better readability and maintainability.

1. Separate Business Logic from Rendering

The core principle is separation of concerns. Business logic—such as which fields are required or when a field should be visible—should live in a schema. The rendering logic simply consumes this schema and renders accordingly.

2. Define Fields Declaratively

Each field is defined as an entry in the schema, where you can specify properties like:

label

type

required

renderWhen (a function that determines if the field should appear)

disableWhen / requiredWhen (functions that determine other dynamic behaviors)

These functions have access to the entire form state, enabling reactive behavior based on other fields.

3. Schema-Driven Rendering
At runtime, the renderer loops through the schema, executes the functions (e.g., renderWhen), and renders only the applicable fields in the specified order. This makes field order predictable and editing the form structure as simple as reordering keys in the schema.

4. Schema Updates for Imperative Changes
In some cases, you need to update fields imperatively—say, after a lookup action completes, you want to disable a field. Rather than mutate the UI directly, the solution is to update the schema itself. For instance, you might set disableWhen to always return true for that field, then re-render the form with the updated schema.

Real-World Use Case: A Data Cataloging Platform
This solution was implemented in a data cataloging platform, where products needed to register with various storage structures and connection details. The form requirements were complex:

Conditional fields based on storage type

Data fetching to populate dropdowns

Field pre-filling based on existing data

Integration with existing UI libraries and form state managers

A schema-based form system handled this complexity elegantly. Fields were grouped logically (e.g., basic details, authentication, database), and each group could have its own renderWhen logic. Nested fields (like database engine details) were defined cleanly under parent groupings.

Handling Grouped Fields and Hierarchical Logic
Grouping fields improves clarity and control. For example, a form could be structured into sections:

Basic Info: Name, description

Authentication: Auth provider, endpoint

Database: Port, engine, endpoint

Even within groups, fields retain their own logic. The group itself can have a renderWhen condition—if it returns false, the entire section is skipped. The field-level logic still applies, resulting in an AND relationship between group and field visibility.

Conclusion
By using a schema-driven approach to form creation, we gain several benefits:

Cleaner, more readable code

Separation of business and rendering logic

Easier updates when requirements change

Built-in support for dynamic and reactive behavior

Declarative grouping and nested logic without clutter

This design scales well in large applications and makes form development significantly more maintainable. If you're building complex forms and tired of wrestling with deeply nested components and imperative hacks, this might be the approach worth trying.