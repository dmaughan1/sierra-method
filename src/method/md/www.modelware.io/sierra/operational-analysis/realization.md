---
template:
  id: https://www.modelware.io/sierra/operational-analysis/realization
  name: "Capability Realization"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
## Capability Realization

Every objective's capability should be realized by an operational entity. This graph
draws the `Objective ▸ requires ▸ Capability ▸ isAssignedTo ▸ Entity` chain. A capability
with no entity edge is drawn in red: it is required by the mission but not yet realized.

```graph
---
layout:
  mode: force
  running: true
  fit: true
  padding: 24
  force:
    repulsion: 4200
    linkDistance: 130
    springStrength: 0.006
    gravity: 0.0012
    damping: 0.90
    maxSpeed: 4
    group:
      intraAttraction: 0.0025
      interRepulsionFactor: 3.0
stylesheet:
  - selector: node [value.includes("objectives#")]
    group: objective
    style:
      fill: yellow
      stroke: grey
      stroke-width: 1
  - selector: node [value.includes("/entities#") || value.includes("/stakeholders#")]
    group: entity
    style:
      fill: lightgreen
      stroke: grey
      stroke-width: 1
  - selector: node [value.includes("capabilities#")]
    group: capability
    style:
      fill: cyan
      stroke: grey
      stroke-width: 1
  - selector: node [value.includes("capabilities#") && !outgoing.some(e => e.value.includes("isAssignedTo"))]
    group: unrealized
    style:
      fill: tomato
      stroke: grey
      stroke-width: 1
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX entity: <https://www.modelware.io/sierra/entity#>

CONSTRUCT {
  ?objective mission:requires ?capability .
  ?capability entity:isAssignedTo ?entity .
}
WHERE {
  ?objective a mission:Objective ;
             mission:requires ?capability .
  OPTIONAL { ?capability entity:isAssignedTo ?entity }
}
```
