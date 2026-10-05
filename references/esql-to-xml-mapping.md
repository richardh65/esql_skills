# ESQL-to-XML Mapping Reference

## Common mappings

| ESQL | XML |
|---|---|
| `InputRoot.XMLNSC.request.operation` | `<request><operation>value</operation></request>` |
| `InputRoot.XMLNSC.request.customer.id` | `<request><customer><id>value</id></customer></request>` |
| `InputRoot.XMLNSC.request.item[1].code` | First `<item>` contains `<code>` |
| `InputRoot.XMLNSC.request.item[2].code` | At least two `<item>` elements; second contains `<code>` |
| `InputRoot.XMLNSC.request.(XMLNSC.Attribute)id` | `<request id="value">...</request>` |

## Important distinction

The following are not XML payload paths:

```text
OutputRoot
OutputLocalEnvironment
Environment
```

They refer to ACE message-tree areas manipulated by the flow.

MQ metadata such as:

```text
InputRoot.MQMD.MsgId
InputRoot.MQMD.CorrelId
InputRoot.MQMD.Format
```

also does not become XML merely because it is referenced by ESQL.

## Suggested test values

Use values that make debugging obvious:

```text
operation = TestOperation
customerId = TEST-001
transactionId = TEST-12345
routingKey = TEST_ROUTE
```

Avoid ambiguous values such as empty strings unless the test specifically concerns empty/missing values.
