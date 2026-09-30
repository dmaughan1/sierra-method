---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Operational Analysis Layer

A small analysis layer over the Fire Force VI Mission Control model. Each section states an
engineering question, answers it with a live query over the model, and reads the result.

The Sierra traceability chain this layer leans on is:

`Concern ▸ derives ▸ Objective ▸ requires ▸ Capability ▸ isAssignedTo ▸ Entity`

and, for behavior, `Process ▸ describes ▸ Capability`.

## Conformance — every capability traces up to an objective

**Rule.** In Sierra a capability exists only to serve the mission, so every `Capability`
must be required by at least one `Objective`. The table flags each capability as conforming
or not.

```table
---
orderBy: "Capability asc"
stylesheet:
  - selector: cell[col === "Conforms" && value === "false"]
    target: value
    style: { color: "#DC2626", font-weight: 600 }
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX base: <https://www.modelware.io/sierra/base#>

SELECT ?Capability ?Description (EXISTS { ?cap mission:isRequiredBy ?o } AS ?Conforms)
WHERE {
  ?cap a mission:Capability .
  BIND(REPLACE(STR(?cap), "^.*[#/]", "") AS ?Capability)
  OPTIONAL { ?cap base:description ?Description }
}
ORDER BY ?Capability
```

All nine capabilities return `Conforms = true`: the model is clean against this rule. The
query would surface any capability invented without a mission justification by printing
`false` in red.

## Near miss — assigned to an entity but no describing process

A capability is *fully operationalized* when it is both **assigned to an entity** (someone
provides it) **and described by a process** (there is a defined behavior for it). A near miss
has the first but is missing the second — it is one step short.

```table
---
orderBy: "Capability asc"
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>
PREFIX process: <https://www.modelware.io/sierra/process#>
PREFIX base: <https://www.modelware.io/sierra/base#>

SELECT ?Capability ?Description
WHERE {
  ?cap a mission:Capability ;
       entity:isAssignedTo ?entity .
  OPTIONAL { ?cap base:description ?Description }
  FILTER NOT EXISTS { ?process process:describes ?cap }
  BIND(REPLACE(STR(?cap), "^.*[#/]", "") AS ?Capability)
}
GROUP BY ?Capability ?Description
ORDER BY ?Capability
```

Six capabilities (C1, C3, C4, C6, C7, C8) have an owning entity but no process describing how
they work — only C2 and C5 are covered by a process (P1 and P2). This differs from the orphan
query below: these capabilities *are* allocated, they just lack a behavioral definition.

## Orphan — capability required by the mission but realized by nobody

The rule tested here is the downward half of the chain: every `Capability` must be
`isAssignedTo` some `Entity`. This section is written as **Question ▸ Evidence ▸
Interpretation**.

### Engineering question

Is every capability the mission requires actually realized by an operational entity, or is
there a capability that nothing in the architecture provides?

### Evidence

```table
---
orderBy: "Capability asc"
stylesheet:
  - selector: cell[col === "Capability"]
    target: value
    style: { color: "#DC2626", font-weight: 600 }
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>
PREFIX base: <https://www.modelware.io/sierra/base#>

SELECT ?Capability ?Description
WHERE {
  ?cap a mission:Capability .
  OPTIONAL { ?cap base:description ?Description }
  FILTER NOT EXISTS { ?cap entity:isAssignedTo ?entity }
  BIND(REPLACE(STR(?cap), "^.*[#/]", "") AS ?Capability)
}
ORDER BY ?Capability
```

### Interpretation

Exactly one orphan: **C9 — onboard AI fire detection**. C9 was added with the onboard-AI
increment and traces up correctly (objective O5 requires it), but no entity has it via
`hasCapability`. That is a real allocation gap: the mission asks for onboard detection and
the architecture currently has nothing that delivers it. An empty result here would have
meant every capability is realized; the red row shows the one that is not.

## Coverage — objectives × entities

How many capabilities connect each mission objective to each operational entity? Missing
combinations are shown as an explicit `0` (via `COALESCE(?n, 0)`) so gaps are visible rather
than absent.

```matrix
---
rowColumnLabel: Objective / Entity
stylesheet:
  - selector: cell [Number(value) === 0]
    style: { background-color: "#fde2e2" }
  - selector: cell [Number(value) > 0]
    style: { background-color: "#d7f5dd" }
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  ?row a mission:Objective .
  ?column a entity:Entity .

  OPTIONAL {
    SELECT ?row ?column (COUNT(DISTINCT ?cap) AS ?n)
    WHERE {
      ?row mission:requires ?cap .
      ?cap entity:isAssignedTo ?column .
    }
    GROUP BY ?row ?column
  }
}
ORDER BY ?row ?column
```

The **O5 row is entirely zero** — no entity contributes to the onboard-detection objective,
the same gap the orphan query found, now visible at the objective level. NetworkInterface and
the actor-entities also show mostly zeros: they each contribute to a single objective.

## Analysis graph — capability realization (reused template)

This graph is the reusable `Capability Realization` template, invoked here and again from the
Operations Dashboard. It shapes the model around one question — realization — instead of
drawing the whole model.

```compose
template: https://www.modelware.io/sierra/operational-analysis/realization
```

## Scripted analysis — realization coverage per objective

Query the objective/capability/entity chain, **compute** the percent of each objective's
capabilities that are realized by an entity, then **render** it as a chart.

```python
include('src/method/py/utils.py')
import micropip
await micropip.install(['matplotlib'])
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

result = await query("""
  PREFIX mission: <https://www.modelware.io/sierra/mission#>
  PREFIX entity: <https://www.modelware.io/sierra/entity#>
  SELECT ?objective ?capability ?entity
  WHERE {
    ?objective a mission:Objective ;
               mission:requires ?capability .
    OPTIONAL { ?capability entity:isAssignedTo ?entity }
  }
""")

# compute: for each objective, share of required capabilities that have an entity
required = {}
realized = {}
for r in result['rows']:
    o = frag(r.get('objective', ''))
    c = r.get('capability', '')
    required.setdefault(o, set()).add(c)
    if r.get('entity'):
        realized.setdefault(o, set()).add(c)

objectives = sorted(required)
pct = [100 * len(realized.get(o, set())) / len(required[o]) for o in objectives]

fig, ax = plt.subplots(figsize=(7, 0.6 * len(objectives) + 1))
colors = ['#2e7d32' if p == 100 else '#c62828' for p in pct]
ax.barh(objectives, pct, color=colors)
ax.set_xlim(0, 100)
ax.invert_yaxis()
ax.set_xlabel('% of required capabilities realized by an entity')
ax.set_title('Capability Realization Coverage per Objective')
for i, p in enumerate(pct):
    ax.text(min(p + 2, 96), i, f'{p:.0f}%', va='center', fontsize=9)
plt.tight_layout()
display(image_html(fig))
```

Four objectives sit at 100%; **O5 is at 0%**, because its only capability (C9) is the orphan.
The derived metric turns the raw orphan fact into a per-objective readiness score.
