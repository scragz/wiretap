# Wiretap Language Specification (v1.1)

## 1. Syntax & Grammar

### 1.1 Lexical Rules

* **Encoding:** UTF-8.
* **Whitespace:** Ignored, except as a delimiter between tokens.
* **Comments:** Start with `#`, extend to End-Of-Line (EOL). Treated as whitespace by the parser.
* **Case Sensitivity:**
* **Identifiers (`<id>`):** Restricted to `[a-z][a-z0-9_]{0,15}` (lowercase snake_case).
* **Keywords/Headers:** UPPERCASE only (`META:`, `NODES:`, `FLOW:`, `MOD:`, `->`, `true`, `false`).
* **Strings:** Case-sensitive.


* **Terminators:** Statements must end with a Newline `\n` or Semicolon `;`.

### 1.2 EBNF Grammar (The Core)

```ebnf
patch        ::= [meta_block] node_block flow_block [mod_block]

meta_block   ::= "META:" eol { meta_stmt eol }
meta_stmt    ::= <id> ":" <value>

node_block   ::= "NODES:" eol { node_decl eol }
node_decl    ::= <id> "=" <prim_ref> "(" [param_list] ")"
prim_ref     ::= <type_id> [ ":" <variant_id> ]
param_list   ::= param_assign { "," param_assign }
param_assign ::= <key_id> ":" <value>

flow_block   ::= "FLOW:" eol { flow_stmt eol }
flow_stmt    ::= endpoint edge endpoint
edge         ::= "->" | "-[" edge_props "]->"

mod_block    ::= "MOD:" eol { mod_stmt eol }
mod_stmt     ::= endpoint edge target_param

endpoint     ::= <node_id> [ "." <port_id> ]
target_param ::= <node_id> "." <param_id>

edge_props   ::= [ prop_list ]
prop_list    ::= prop_val { "," prop_val }
prop_val     ::= <float> | <tag_id>

value        ::= <float> | <bool> | <string> | <list>

```

### 1.3 Section Rules

1. **META:** Optional. If present, must be first. Contains key-value metadata.
2. **NODES:** Required. Defines all graph nodes.
3. **FLOW:** Required. Defines signal routing (Audio/CV/Gate).
4. **MOD:** Optional. Defines parameter modulation.

---

## 2. Semantics & Resolution

### 2.1 Identifier Resolution

* **Node Scope:** IDs must be unique within the `patch`.
* **Port/Param Scope:** `<id>` resolves against the **Primitive Registry** definition for that node's `type:variant`.
* **Endpoint Resolution:**
* **Strict Mode:** Endpoints must be fully qualified (`node.port`).
* *Exception:* Ports marked `default: true` in the Registry may be omitted.


* **Mod Source:** Must resolve to a **Port** (Output).
* **Mod Target:** Must resolve to a **Parameter**.



### 2.2 Edge Semantics & Properties

The syntax `src -[props]-> dst` is unified for both Flow and Mod.

| Context | Property | Interpretation |
| --- | --- | --- |
| **FLOW** | Float | **Gain** (Linear attenuation). Default `1.0`. |
| **FLOW** | `fb` | **Feedback Safety Flag**. Acknowledges a cycle. |
| **FLOW** | `mix` | **Intentional Sum**. Suppresses implicit-sum warnings. |
| **MOD** | Float | **Depth**. Scalar multiplier. Default `1.0`. |
| **MOD** | `fb` | (Rare) Feedback modulation flag. |

### 2.3 Modulation Transfer Function

Modulation edges apply the following function:

* : Source signal value (Volts or Normalized).
* **Curve:** Transform applied to input (Linear default).
* **Depth:** Float scalar (Attenuverter).
* **Offset:** Bias added post-attenuation.
* **Clamp:** Hard limits defined by the Target Parameter's `min`/`max`.

---

## 3. The Primitive Registry (Source of Truth)

Implementations must validate against a schema-defined Registry.

### 3.1 Registry Schema

Every primitive definition MUST include:

1. **Ports:** Map of `<id>` to definition.
* `dir`: `in` | `out`
* `sig`: `AUDIO` | `CV` | `GATE` | `TRIG` | `PITCH`
* `default`: `true` (optional, max 1 per direction)


2. **Params:** Map of `<id>` to definition.
* `type`: `float` | `int` | `bool` | `enum` | `string`
* `unit`: `hz`, `ms`, `v`, `norm` (normalized 0-1), `ratio`
* `min`, `max`, `default`


3. **Variants (optional):** If a primitive has structural variants (e.g., `filter:lp` vs `filter:state_variable`), they are defined as separate entries or sub-entries.

### 3.2 Example Registry Entry

```json
"filter": {
  "ports": {
    "in":     { "dir": "in",  "sig": "AUDIO", "default": true },
    "out":    { "dir": "out", "sig": "AUDIO", "default": true },
    "cutoff": { "dir": "in",  "sig": "CV" }
  },
  "params": {
    "freq":   { "type": "float", "unit": "hz", "default": 1000, "min": 20, "max": 20000 },
    "res":    { "type": "float", "unit": "norm", "default": 0.0, "min": 0, "max": 1.0 }
  }
}

```

---

## 4. Safety & Validation Policies

### 4.1 Signal Flow Safety

* **Directionality:** `out` -> `in` only. `in` -> `out` is a Critical Error.
* **Type Safety:**
* **Strict Mode:** `sig` types must match exactly.
* **Loose Mode:** `AUDIO`/`CV` may mix with Warning. `GATE`/`TRIG` may mix.


* **Implicit Summing:**
* **Rule:** Fan-in > 1 is strictly allowed.
* **Policy:** Raw Sum.
* **Warning:** Validator emits `WARN` on fan-in > 1 unless the edge contains the `mix` tag (e.g., `osc2 -[mix]-> filt.in`).



### 4.2 Feedback Safety

* **Definition:** A cycle is any directed path .
* **Requirement:** Every cycle must contain at least one edge explicitly tagged `fb`.
* **Error Level:**
* Strict Mode: **ERROR** (Parse fails).
* Loose Mode: **WARN**.


* **Gain Warning:** Warn if a feedback loop has no attenuation edges (Gain < 1.0) and no damping filter nodes.

### 4.3 Versioning

* **Format:** SemVer (`MAJOR.MINOR`).
* **Header:** `META: lang: wiretap@1.1`
* **Policy:**
* **Diff Major:** Critical Error (Reject).
* **Diff Minor (Parser < File):** Warning. Best-effort parse.
* **Diff Minor (Parser > File):** Accept (Backward compatible).



---

## 5. Event & Time Model

To ensure cross-rig portability:

1. **Logic Thresholds:**
* **High (Gate Open):** Signal > `0.5` (Normalized) or `1.0V` (Volts).
* **Low (Gate Closed):** Signal ≤ Threshold.


2. **Trigger Definition:**
* A "Trigger" is recognized on the **Rising Edge** (crossing the threshold from Low to High).
* Pulse width is implementation-dependent but must satisfy the Rising Edge requirement.


3. **Clocking:**
* Clock inputs respond to Triggers (Rising Edge).
* Reset inputs are **Asynchronous** (immediate interrupt).



---

## 6. Canonicalization Rules

For tool interoperability (`Text` -> `AST` -> `Text`):

1. **Section Headers:** Always present, uppercase.
2. **Sort Order:**
* `META`: Key-alphabetical.
* `NODES`: ID-alphabetical.
* `FLOW/MOD`: Source ID, then Dest ID.


3. **Formatting:**
* Floats: `0.5` (not `.5`), max 4 decimals.
* Whitespace: Single space after delimiters.
* Expansions: All chains (`A->B->C`) and fan-outs (`{A,B}`) are expanded to single statements.



---

## 7. Example: Validated v1.1 Patch

```text
META:
  lang: flowh@1.1
  name: "Dub Loop"

NODES:
  clk   = clock:master (bpm:140)
  delay = delay:tape   (time:350ms, feed:0.0)
  dub   = fx:filter    (freq:400hz)
  kick  = drum:kick    ()
  mix   = mix:sum      ()
  out   = out:stereo   ()

FLOW:
  clk.out -> kick.trig_in
  kick.out -> mix.in
  mix.out -> out.L
  mix.out -> out.R

  # Send to Delay
  mix.out -[0.5]-> delay.in

  # Feedback Loop (Marked)
  delay.out -> dub.in
  dub.out -[fb, 0.8]-> delay.in

MOD:
  # No modulation in this patch

```
