---
name: ace-esql-xml-test-message
description: Build and explain XML test messages for IBM App Connect Enterprise (ACE) or IBM Integration Bus (IIB) message flows so a specific ESQL statement can be exercised through a message queue. Use this skill when a user provides ESQL and wants to know what XML message should be sent to MQ to make a particular line execute, populate an OutputLocalEnvironment/Environment value, route a message, access an XMLNSC field, or otherwise test a specific message-tree expression.
---

# ACE ESQL XML Test Message Builder

## Purpose

Help the user construct a realistic XML message that, when sent through an IBM App Connect Enterprise (ACE) / IBM Integration Bus (IIB) flow, creates the message-tree structure expected by a specific ESQL statement.

The primary goal is not merely to produce syntactically valid XML. The goal is to map:

1. The ESQL expression
2. The required ACE message-tree path
3. The corresponding XML structure
4. A complete test message suitable for sending to the flow's MQ input
5. The expected result after the statement executes

# Instructions
1. When this skill is triggered, ask the user which .esql file holds the ESQL code.
2. When this skill is triggered, ask the user which .msgflow file holds the MSGFLOW code.
2. When this skill is triggered, ask the user what line number fron the ESQL file that want to test.

## Core workflow

When the user supplies a line number from an ESQL file:

### 1. Identify the target expression

Parse the statement and identify every input-tree value it reads.

For example:

```esql
SET OutputLocalEnvironment.Destination.RouterList.DestinationData[1].labelname =
    InputRoot.XMLNSC.request.operation;
```

The input dependency is:

```text
InputRoot
└── XMLNSC
    └── request
        └── operation
```

Therefore the XML must contain:

```xml
<request>
    <operation>...</operation>
</request>
```

### 2. Distinguish input from output paths

Treat paths beginning with these as output/working-tree paths unless the statement explicitly reads them:

- `OutputRoot`
- `OutputLocalEnvironment`
- `OutputEnvironment`
- `Environment`
- `LocalEnvironment`

Treat paths beginning with these as likely input dependencies:

- `InputRoot`
- `InputLocalEnvironment`

Do not add XML elements merely because they appear on the left-hand side of an assignment.

### 3. Determine the message domain

Pay attention to the parser/domain used by the flow.

Common examples:

```esql
InputRoot.XMLNSC
InputRoot.JSON.Data
InputRoot.MQMD
InputRoot.BLOB.BLOB
```

If the ESQL accesses `InputRoot.XMLNSC`, produce XML unless the user indicates a different parser/message-domain setup.

If the parser/domain is uncertain, explicitly state the assumption.

### 4. Map ESQL paths to XML

For XMLNSC, map the logical ESQL path to XML elements.

Example:

```esql
InputRoot.XMLNSC.request.operation
```

maps to:

```xml
<request>
    <operation>VALUE</operation>
</request>
```

Array/subscript examples require special attention.

For:

```esql
InputRoot.XMLNSC.request.items.item[2].code
```

produce:

```xml
<request>
    <items>
        <item>
            <code>...</code>
        </item>
        <item>
            <code>VALUE_FOR_SECOND_ITEM</code>
        </item>
    </items>
</request>
```

Do not confuse ESQL array indexing with an XML element literally named `[2]`.

### 5. Account for namespaces

If the ESQL uses namespace-qualified names or the flow expects a namespace, preserve it.

For example, if the ESQL is effectively accessing a namespace-qualified element, produce an appropriate namespace declaration:

```xml
<request xmlns="http://example.com/request">
    <operation>TestOperation</operation>
</request>
```

If the namespace requirement cannot be established from the supplied ESQL, say so rather than inventing one.

### 6. Account for XML attributes

If the ESQL accesses an XML attribute, the test message must contain an attribute rather than a child element.

For example:

```esql
InputRoot.XMLNSC.request.(XMLNSC.Attribute)operation
```

requires something like:

```xml
<request operation="TestOperation"/>
```

Do not incorrectly generate:

```xml
<request>
    <operation>TestOperation</operation>
</request>
```

### 7. Account for repeated elements

If the ESQL uses a repeated XML element or indexed reference, create enough occurrences for the requested index.

For example:

```esql
InputRoot.XMLNSC.orders.order[3].id
```

requires at least three `<order>` elements.

### 8. Account for predicates and selectors

If the ESQL uses predicates, filters, or selectors, construct XML satisfying the predicate.

For example:

```esql
InputRoot.XMLNSC.orders.order[
    type = 'priority'
].id
```

must include an order whose `type` is `priority`.

Provide the minimum useful XML necessary to satisfy the selector.

### 9. Consider flow-level transformations

If the user provides surrounding ESQL, node configuration, or flow information, determine whether an earlier node modifies the message tree before the target statement executes.

Do not assume the MQ payload arrives unchanged at the target Compute/Route node.

When relevant, explain:

```text
MQ message
    ↓
MQInput parser
    ↓
earlier node(s)
    ↓
target node
    ↓
target ESQL statement
```

The test XML should represent the message entering the flow, not necessarily the exact tree immediately before the target statement, unless the user explicitly asks for the latter.

### 10. Produce a minimal test message first

Prefer a minimal XML payload that contains only the elements needed to exercise the target statement.

Then optionally provide a more realistic example if useful.

Use explicit test values that make the expected result obvious.

For example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<request>
    <operation>TestOperation</operation>
</request>
```

### 11. Explain the expected result

For an assignment such as:

```esql
SET OutputLocalEnvironment.Destination.RouterList.DestinationData[1].labelname =
    InputRoot.XMLNSC.request.operation;
```

explain that with:

```xml
<operation>TestOperation</operation>
```

the expected value becomes:

```text
OutputLocalEnvironment.Destination.RouterList.DestinationData[1].labelname
    = "TestOperation"
```

Where useful, show the relevant message-tree structure before and after the statement.

## Important ACE/IIB distinctions

### XML element vs XML attribute

Clearly distinguish:

```xml
<operation>ABC</operation>
```

from:

```xml
<request operation="ABC"/>
```

based on the ESQL path.

### LocalEnvironment vs Environment

Do not describe `OutputLocalEnvironment` as part of the incoming XML. It is an ACE message-tree area used by the flow.

### RouterList and DestinationData

For statements involving:

```text
OutputLocalEnvironment.Destination.RouterList.DestinationData[n]
```

explain that these values are routing metadata rather than XML payload fields.

Do not add `Destination`, `RouterList`, or `DestinationData` elements to the XML unless the ESQL is actually reading those structures from an input tree.

### MQMD

Do not invent MQMD fields in the XML. If a statement reads `InputRoot.MQMD`, identify the requirement separately because MQMD is MQ message metadata, not XML payload.

### XMLNSC parser

If the target expression starts with `InputRoot.XMLNSC`, assume the MQInput node is configured to parse the payload as XML/XMLNSC unless the user indicates otherwise.

If parser configuration is important to the test, call it out explicitly.

## Handling incomplete ESQL

If the supplied line is insufficient to determine the exact XML structure, do not guess silently.

Explain what can be determined and request the smallest missing piece.

Useful questions include:

- What is the complete ESQL statement?
- Is the input parsed as XMLNSC?
- What is the XML root element?
- Is there a namespace?
- Are there earlier Compute/Mapping nodes that modify the message?
- What is the MQInput parser/domain configuration?

If a reasonable assumption can be made, proceed with the assumption and clearly label it.

## Output format

When enough information is available, structure the response as:

### 1. ESQL being tested

Quote the user's ESQL briefly.

### 2. Required input path

Show the relevant message-tree path:

```text
InputRoot
└── XMLNSC
    └── request
        └── operation
```

### 3. Minimal XML message

Provide the complete XML payload in a code block.

### 4. Why this works

Explain how each XML element maps to the ESQL expression.

### 5. Expected result

Show the value that should be produced or changed in the ACE message tree.

### 6. MQ/ACE test notes

Mention parser/domain, namespaces, headers, or flow configuration only when relevant.

## Test-message quality rules

Every generated XML message should:

- Be well-formed XML.
- Contain the minimum required structure.
- Use values that clearly demonstrate the expected result.
- Include an XML declaration when the message is intended to be copied directly into an MQ test tool.
- Preserve required namespaces.
- Include enough repeated elements for any requested index.
- Never fabricate unrelated business fields unless the user asks for a realistic sample.
- Clearly distinguish assumptions from facts.

## Advanced mode: multiple statements

If the user provides multiple ESQL statements, identify the union of their input dependencies and generate one XML message capable of exercising all of them.

Also identify which statements cannot be exercised by payload XML alone, such as statements depending on:

- MQMD fields
- LocalEnvironment populated by an earlier node
- Environment variables
- external resources
- shared variables
- database lookups
- flow configuration

## Troubleshooting mode

If the user says the generated message did not cause the ESQL line to execute or did not produce the expected value:

1. Re-check the exact ESQL path.
2. Check XML root and element names.
3. Check namespaces.
4. Check repeated-element indexing.
5. Check XMLNSC parser configuration.
6. Check whether an earlier node transforms the tree.
7. Check whether the ESQL is guarded by an IF/CASE/PROPAGATE/filter condition.
8. Check whether the statement depends on LocalEnvironment, Environment, MQMD, or external data.
9. Suggest a trace/debug approach if necessary.

Prefer explaining the message-tree state immediately before the target statement.

## Example

Input:

```esql
SET OutputLocalEnvironment.Destination.RouterList.DestinationData[1].labelname =
    InputRoot.XMLNSC.request.operation;
```

Recommended response:

```text
Required input path:

InputRoot
└── XMLNSC
    └── request
        └── operation

Minimal MQ XML payload:

<?xml version="1.0" encoding="UTF-8"?>
<request>
    <operation>TestOperation</operation>
</request>
```

Expected result:

```text
OutputLocalEnvironment.Destination.RouterList.DestinationData[1].labelname
= "TestOperation"
```

Explain that `<operation>` is required because the right-hand side reads `InputRoot.XMLNSC.request.operation`. The `OutputLocalEnvironment...` path is created/populated by ESQL and should not be represented as XML in the MQ payload.
