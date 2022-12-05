---
layout: post
title: How to adopt FHIR Terminology in your Python data analysis toolbox.
---

# Using FHIR Terminology your Python workflow

## Introduction

- FHIR Terminology has some wonderfull features but as a data scientist it's hard to use it in my analysis workflow.
- First of all, FHIR Terminology is a REST API and this means lots of [HTTP calls using the `requests` library](https://requests.readthedocs.io/en/latest/). Calls to a REST API involve some technical details that do not contribute to the data analysis we are making. It's only polluting the code readability.
- Secondly, FHIR Resources are tranfered in JSON and in Python we decode this to dictionaries. Dictionaries are usefull for small temporary data structures but FHIR Resources are way to big to put in a dictionary. I don't know by heart which keys their are available in a ValueSet dictionary. Instead, we want to have dedicated class objects with attributes that help us accessing the attributes we need.

**If we want to integrate FHIR Terminology into our workflow, we need validated typed objects for all the FHIR resources and elements related to terminology. And we want operations on those elements to translate to Python operators or functions.**

## Important aspects FHIR Terminology

Before goinging into the details of how we handle terminology in Python, let's discuss the most important data structures related FHIR Terminology

### Coding

A Coding is a the most basic structure refering to a single code that is part of a CodeSystem. An example of such a code is [`386661006 |Fever|` from SNOMED-CT](https://browser.ihtsdotools.org/?perspective=full&conceptId1=386661006&edition=MAIN&release=&languages=en). This can be respresented using a FHIR Coding:

```fhir+json
{
    system: "http://snomed.info/sct",
    code: "386661006",
    display: "Fever"
}
```

More info on the meaning of the individual fields in the [FHIR documentation](https://build.fhir.org/datatypes.html#Coding).

### CodeableConcept

Often it's hard to represent the meaning of reallife medical concepts using exact _Codings_ and you want to combine it with some nuanced text. This is were the **CodeableConcept** comes in:

```fhir+json
{
    text: "Light fever",
    coding: [
       {
            system: "http://snomed.info/sct",
            code: "386661006",
            display: "Fever"
        }
    ]
}
```

More info on the meaning of the individual fields in the [FHIR documentation](https://build.fhir.org/datatypes.html#CodeableConcept).

### CodeSystem

#### Operations:

- subsumtion
- validate-code
- lookup

### ValueSet

#### Operations:

- expand
- validate-code

## Parsing data using Pydantic

## Operations as operators

```python
fhir_tx_server = new FHIRTerminologyServer("<base-url-to-fhir-terminology>")
sct =  new CodeSystem.from_canonical("http://snomed.info/sct", server=fhir_tx_server)
chronic_fever = new SCTCoding("704425001 |Chronic fever|")

assert chonric_fever in sct, "Expected 'Chornic fever' validate as code part of SNOMED-CT"
```

```python
sct_fever = new SCTCoding("386661006 |Fever|")
sct_chronic_fever = new SCTCoding("704425001 |Chronic fever|")

assert sct_chronic_fever < sct_fever, "Expect 'Chronic fever' to be a child of 'Fever'"
```
