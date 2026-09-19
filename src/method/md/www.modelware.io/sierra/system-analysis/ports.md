---
template:
  id: https://www.modelware.io/sierra/system-analysis/ports
  name: "Ports"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---
# Ports

Define the **ports** of components — the interaction points through which a component
exchanges signals, material, or energy with the rest of the system — and declare the
**direction** in which each port carries an item.

**Focal type.** `component:Port` — an interaction point of a component, specialized from
`base:Contained` and `base:Element`.

**Structure.** A port is attached to a component through the `component:hasPort` relation.
Because `hasPort` is declared `inverse functional`, each port has at most one owning
component (equivalently, its reverse `component:portOf` is functional). Each port also
carries an item in a single direction, given by the `component:direction` property, whose
range is the `component:Direction` scalar — an enumeration whose only permitted values are
`In` and `Out`.

**Expected content.** For each port: its **direction** (`In` or `Out`) and, optionally, a
free-text **description**. The owning **component** is shown read-only for context — ports
are attached to their component on the *Components* page, and this page focuses on typing
each port's direction.

**Rule.** Every port must declare a direction. A port with no direction is under-specified:
downstream connection and power-flow analyses cannot tell whether it is a source or a sink,
so the page flags it. The permitted values (`In`, `Out`) are enforced by the
`component:Direction` scalar itself, so the shape only needs to check that a direction is
present.

```table-editor
---
columns: { this: { label: "Port" } }
stylesheet:
  - selector: cell[col === "Direction" && value]
    target: value
    style:
      padding: 4px 12px
      border-radius: 999px
      font-size: 12px
      font-weight: 600
      color: "#ffffff"
  - selector: cell[value === "Out"]
    target: value
    style:
      background-color: "#2563EB"
  - selector: cell[value === "In"]
    target: value
    style:
      background-color: "#10B981"
---
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

component:PortShape
    a sh:NodeShape ;
    sh:targetClass component:Port ;
    sh:sparql [
        sh:message "Every port must declare a direction (In or Out)." ;
        sh:select """
            SELECT $this WHERE {
                $this a component:Port .
                FILTER NOT EXISTS { $this component:direction ?d }
            }
        """ ;
    ] ;
    sh:property [
        sh:path component:portOf ;
        sh:name "Component" ;
        sh:class component:Component ;
        sh:maxCount 1 ;
        dash:readOnly true ;
        sh:order 0 ;
    ] ;
    sh:property [
        sh:path component:direction ;
        sh:name "Direction" ;
        sh:maxCount 1 ;
        sh:order 1 ;
    ] ;
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
        sh:order 2 ;
    ] ;
    .
```
