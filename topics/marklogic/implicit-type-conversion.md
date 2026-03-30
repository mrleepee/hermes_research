# Implicit Type Conversion in MarkLogic

## Overview

MarkLogic implements implicit (automatic) type conversion according to the **XQuery/XPath Data Model (XDM)** specification, with some MarkLogic-specific behaviors. Understanding these rules is essential for writing correct queries and avoiding unexpected results.

---

## The XQuery Data Model (XDM)

Before understanding conversion, you need to understand the type system. XDM defines a strict type hierarchy:

### Atomic Types (Selected)

```
anyAtomicType (abstract)
├── xs:string
├── xs:boolean
├── xs:decimal
│   ├── xs:integer
│   └── (numeric subtypes)
├── xs:float
├── xs:double
├── xs:date
├── xs:dateTime
├── xs:time
├── xs:duration
├── xs:yearMonthDuration
├── xs:dayTimeDuration
├── xs:hexBinary
├── xs:base64Binary
├── xs:anyURI
├── xs:QName
├── xs:NOTATION
└── xs:untypedAtomic
```

### Key Types Explained

| Type | Description |
|------|-------------|
| `xs:untypedAtomic` | Result of parsing untyped data (e.g., from XML text nodes) |
| `xs:untyped` | Type of elements with no type annotation in schema-less documents |
| `xs:anyURI` | URI values |
| `xs:hexBinary` | Hex-encoded binary |
| `xs:base64Binary` | Base64-encoded binary |
| `xs:NOTATION` | XML notation (rarely used) |

---

## What is Implicit Conversion?

Implicit conversion (also called "type promotion" or "coercion") occurs when the XQuery processor automatically converts a value from one type to another without explicit casting. This happens during:

1. **Function argument binding** — passing arguments to functions
2. **Operator evaluation** — using operators like `+`, `=`, `eq`, etc.
3. **Atomization** — extracting atomic values from nodes
4. **Comparison operations** — comparing values of different types
5. **Return type expectations** — function return expressions

---

## Conversion Rules

### 1. Numeric Type Promotion

When an `xs:integer` is used where `xs:decimal`, `xs:float`, or `xs:double` is expected:

```xquery
(: xs:integer promotes to xs:decimal :)
let $x := 5                    (: xs:integer :)
let $y := 3.14                (: xs:decimal :)
return $x + $y                (: Result: 8.14, xs:decimal :)

(: xs:float promotes to xs:double :)
let $a := 1.5e0 cast as xs:float
let $b := 2.5e0               (: xs:double :)
return $a + $b                (: Result: 4.0e0, xs:double :)
```

**Promotion chain:** `xs:integer` → `xs:decimal` → `xs:float` → `xs:double`

### 2. URI Promotion

`xs:string` can promote to `xs:anyURI` in certain contexts:

```xquery
let $base := "http://example.com/"         (: xs:string :)
let $path := "data/orders.xml"             (: xs:string :)
return concat($base, $path)                 (: Returns xs:anyURI in URI contexts :)
```

### 3. UntypedAtomic Conversion

`xs:untypedAtomic` is the type of **atomic values that have not been assigned a more specific type**. It occurs when:
- Atomizing text nodes from schema-less XML documents
- Parsing strings without a known type
- Certain API returns (e.g., some ODATA or JSON conversions)

`xs:untypedAtomic` values are automatically converted to the target type when used:

```xquery
(: === MOST COMMON CASE: Element to String Conversion === :)

(: Element node bound to variable - passed to string-expecting function :)
let $name := <customer>Acme Corp</customer>  (: $name is element customer :)
return concat("Customer: ", $name)            (: "Customer: Acme Corp" :)
                                               (: No .text() needed! :)
                                               (: Element → atomization → xs:untypedAtomic → xs:string :)

(: Same thing with || operator :)
<greeting>Hello</greeting> || ", world"      (: "Hello, world" :)
                                               (: Element atomizes to xs:untypedAtomic, converts to xs:string :)

(: Attributes also atomize and convert :)
let $qty := doc('<order qty="100"/>')/order/@qty
return concat("Quantity: ", $qty)             (: "Quantity: 100" :)
```

**Common untyped conversions:**

```xquery
(: String to numeric :)
xs:untypedAtomic("42") + 1                 (: xs:integer: 43 :)

(: String to boolean :)
xs:untypedAtomic("true") eq true()          (: Converts "true" to xs:boolean: true :)

(: String to dateTime :)
xs:untypedAtomic("2024-01-15T10:30:00") cast as xs:dateTime
```

### 4. String to Boolean Conversion

**Warning:** String `"true"` or `"false"` does **NOT** automatically convert to boolean:

```xquery
(: WRONG - this does NOT work as expected :)
let $flag := "true"
return if ($flag) then "yes" else "no"      (: Error: can't use string in if condition :)

(: CORRECT - explicit casting :)
let $flag := "true" cast as xs:boolean
return if ($flag) then "yes" else "no"      (: Returns: "yes" :)
```

**Valid string-to-boolean contexts:** Only through explicit `cast as xs:boolean` or `xs:boolean()` constructor.

### 5. Empty Sequences

Empty sequences (`()`) promote to any nullable type:

```xquery
(: Empty sequence matches any type :)
let $empty := ()
return if (exists($empty)) then "has value" else "empty"
                                              (: Returns: "empty" :)

(: Used as default values :)
declare function local:or-default($val as xs:string?, $default as xs:string) as xs:string {
  if (empty($val)) then $default else $val
};

local:or-default((), "N/A")                  (: Returns: "N/A" :)
```

### 6. Node Atomization

Atomization extracts the atomic value from a node, which may trigger conversion:

```xquery
(: Given: <price>19.99</price> in a schema-less document :)
doc("sales.xml")/item/price + 0.01           (: Atomizes to xs:untypedAtomic("19.99"), 
                                                 then promotes to xs:decimal: 20.00 :)
```

---

## Comparison Operators and Type Conversion

### Value Comparisons (`=`, `!=`, `<`, `>`, `<=`, `>=`)

Value comparisons attempt type conversion **both ways**:

```xquery
(: Numeric comparison - string "10" promotes to numeric :)
"10" = 10                                   (: true, xs:integer 10 :)

(: String comparison - both converted to xs:string :)
"hello" = "hello"                           (: true :)

(: Duration comparison :)
xs:dayTimeDuration("PT2H") > xs:dayTimeDuration("PT1H")
                                              (: true :)
```

### General Comparisons (`eq`, `ne`, `lt`, `gt`, `le`, `ge`)

These are used for single values and have stricter type requirements:

```xquery
(: Must be same type or promotable :)
xs:integer(5) eq xs:integer(5)              (: true :)
xs:integer(5) eq 5.0                        (: true - numeric promotion :)

(: Different types that can't compare :)
(: xs:date("2024-01-01") eq xs:dateTime("2024-01-01T00:00:00")  -- Error unless cast :)
```

### Node Comparisons (`is`, `<<`, `>>`)

These do **NOT** perform type conversion — they compare node identity and document order only.

---

## Function Argument Coercion

When you call a function, arguments are converted to the declared parameter type:

```xquery
declare function local:double($n as xs:integer) as xs:integer {
  $n * 2
};

local:double("42")                          (: String "42" promoted to xs:integer: 84 :)
```

### Built-in Function Behaviors

Some functions have special conversion rules:

```xquery
(: concat() accepts any type, converts to string :)
concat("Price: ", 19.99, " EUR")            (: "Price: 19.99 EUR" :)

(: count() atomizes and counts :)
count(<items><item/><item/><item/></items>)  (: Returns: 3 :)

(: string() converts anything to string :)
string(42.5)                                (: "42.5" :)
string(true())                              (: "true" :)
string(xs:date("2024-01-15"))               (: "2024-01-15" :)
```

---

## MarkLogic-Specific Behaviors

### 1. Semantics URI Resolution

MarkLogic may automatically resolve relative URIs to absolute URIs:

```xquery
(: In sem:sparql() or sem:triple() contexts :)
(: String URIs may be promoted to sem:uri :)
let $uri := "http://example.com/person/1"
return sem:iri($uri)                        (: Explicit conversion recommended :)
```

### 2. JSON Node Types

When working with JSON in MarkLogic:

```xquery
(: JSON null is not the same as empty sequence :)
xdmp:to-json(map:map() ! map:get(., "missing"))
                                           (: Returns: null (as xs:untypedAtomic) :)

(: Converting JSON numbers :)
js-eval("10.5") + 0.5                      (: May need explicit xs:decimal() conversion :)
```

### 3. Server-Side JavaScript Coercion

MarkLogic's JavaScript API has different coercion rules:

```javascript
// JavaScript coercion (different from XQuery!)
var x = "5";
var result = x + 10;                        // Returns "510" (string concatenation)

// XQuery conversion
xs:integer("5") + 10                        // Returns 15 (numeric addition)
```

---

## Common Pitfalls

### Pitfall 1: String Concatenation vs Addition

```xquery
(: What you might expect :)
"03" + "04"                                 (: NOT "07" - this is numeric addition! :)

(: Result: 7 (xs:integer) - strings converted to integers and added :)

(: For string concatenation, use concat() or || :)
concat("03", "04")                          (: "0304" :)
"03" || "04"                                (: "0304" (XQuery 3.0+) :)
```

### Pitfall 2: Mixed Type Arithmetic

```xquery
(: xs:decimal + xs:double = xs:double :)
1.5 + 2.5e0                                 (: Result: 4.0e0 (xs:double) :)

(: But: xs:integer + xs:decimal = xs:decimal :)
5 + 3.14                                    (: Result: 8.14 (xs:decimal) :)
```

### Pitfall 3: Boolean "True" String

```xquery
(: This is NOT a boolean comparison :)
$myElement eq "true"                        (: Compares string to string! :)

(: Correct approach :)
xs:boolean($myElement)                      (: Explicit conversion :)
```

### Pitfall 4: Untyped Data in Arithmetic

```xquery
(: From schema-less XML: <amount>100</amount> :)
doc("orders.xml")//amount + 50              (: Works: converts untyped to numeric :)

(: But from mixed content: <price>Total: 100</price> :)
doc("mixed.xml")//price + 50                (: Error: can't convert "Total: 100" to number :)
```

---

## Conversion Functions Reference

| Function | Purpose |
|----------|---------|
| `xs:integer()` | Cast to integer |
| `xs:decimal()` | Cast to decimal |
| `xs:float()` | Cast to float |
| `xs:double()` | Cast to double |
| `xs:string()` | Cast to string |
| `xs:boolean()` | Cast to boolean |
| `xs:date()` | Cast to date |
| `xs:dateTime()` | Cast to dateTime |
| `xs:duration()` | Cast to duration |
| `xs:anyURI()` | Cast to URI |
| `xs:untypedAtomic()` | Cast to untypedAtomic |
| `cast as` | Typed conversion expression |
| `treat as` | Asserted conversion (assertion failure if wrong type) |

---

## Best Practices

1. **Be explicit when it matters** — Use `cast as` or constructor functions when conversion behavior matters
2. **Don't rely on implicit conversion** — It can make code harder to understand
3. **Schema-validate when possible** — Schemas make types explicit and predictable
4. **Handle empty sequences explicitly** — Use `if (empty($x)) then ...` rather than relying on implicit rules
5. **Test with real data** — Untyped data can behave unexpectedly

---

## Summary Table

| From | To | Implicit? | Notes |
|------|----|-----------|-------|
| `xs:integer` | `xs:decimal` | Yes | Always |
| `xs:integer` | `xs:float` | Yes | Via decimal |
| `xs:integer` | `xs:double` | Yes | Via decimal |
| `xs:decimal` | `xs:float` | Yes | |
| `xs:decimal` | `xs:double` | Yes | |
| `xs:float` | `xs:double` | Yes | |
| `xs:untypedAtomic` | `xs:string` | Yes | In string contexts |
| `xs:untypedAtomic` | `xs:boolean` | **No** | Must cast explicitly |
| `xs:untypedAtomic` | `xs:numeric` | Yes | When operand is numeric |
| `xs:string` | `xs:boolean` | **No** | Must cast explicitly |
| `xs:string` | `xs:numeric` | **No** | Must cast or use `xs:numeric()` |
| `xs:string` | `xs:anyURI` | In URI contexts | Generally recommended to cast |
| Empty `()` | Any nullable type | Yes | Matches any type |

---

## References

- [XQuery 3.0 Specification - Type Promotion](https://www.w3.org/TR/xquery-30/#dt-type-promotion)
- [XQuery 3.0 Specification - Function Coercion](https://www.w3.org/TR/xquery-30/#dt-function-coercion)
- [XQuery and XPath Data Model 3.0](https://www.w3.org/TR/xpath-datamodel-30/)
- [MarkLogic XQuery Functions Reference](https://docs.marklogic.com/)

---

*Generated by Hermes Research Agent*
