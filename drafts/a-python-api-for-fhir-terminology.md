---
title: ""
layout: post
permalink: /drafts/fhir-tx-python
---

## Using FHIR Terminology your Python workflow

[FHIR Terminology](https://hl7.org/fhir/terminology-module.html) is a powerful tool for healthcare innovators and software developers, but it can be difficult to integrate into a Python data analysis workflow. One issue is that FHIR Terminology is a REST API, which means that using it often involves making a lot of HTTP calls using a library like [requests](https://requests.readthedocs.io). These calls can add technical details to your code that don't contribute to your analysis, and can make your code harder to read.

Another problem is that [FHIR Resources are transferred in JSON format](https://hl7.org/fhir/json.html) (or XML), which means that in Python, you need to decode them into dictionaries. While dictionaries are useful for small data structures, FHIR Resources can be quite large, and working with dictionaries can make it hard to access the data you need.

To make it easier to use FHIR Terminology in your workflow, you need a way to work with **validated, typed objects** for all the FHIR resources and elements related to terminology. And you want operations on these elements to be easy to perform using **Python operators** or functions.

In this blog post, we'll explore some ways to integrate FHIR Terminology into your Python workflow, and make it easier to use in your data analysis projects.

### Key data structures and operations in FHIR Terminology

> Skip this section if you are familiar with the FHIR Terminology specification.

FHIR Terminology is a complex topic, with many different data structures and operations that are important to understand. In this section, we'll go over some of the most important aspects of FHIR Terminology, and how they can be used in your Python workflow.

#### Coding

A Coding is the most basic structure used in FHIR Terminology. It refers to a single code that is part of a CodeSystem. For example, the SNOMED-CT code for "fever" is `386661006 |Fever|`, and it can be represented using a FHIR Coding like this:

```json
{
  "system": "http://snomed.info/sct",
  "code": "386661006",
  "display": "Fever"
}
```

You can find more information about the fields in a Coding in the FHIR documentation.

#### CodeableConcept

Sometimes it's difficult to represent the meaning of real-world medical concepts using exact Codings, and you need to combine them with some additional text. That's where the CodeableConcept comes in. It allows you to combine a Coding with a free-text description, like this:

```
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

You can find more information about the fields in a CodeableConcept in the FHIR documentation.

#### CodeSystem

A CodeSystem is a collection of codes that can be used to represent concepts in a particular domain. For example, SNOMED-CT is a CodeSystem that contains codes for medical concepts.
CodeSystem resources are a little too large to embed in this, but there are some intersting examples on the FHIR website: [AllergyIntoleranceCertainty](https://terminology.hl7.org/CodeSystem-reaction-event-certainty.json.html), [SNOMED-CT](https://terminology.hl7.org/CodeSystem-v3-snomed-CT.json.html), [LOINC](https://terminology.hl7.org/CodeSystem-v3-loinc.json.html), [ICD-19](https://terminology.hl7.org/CodeSystem-icd10.json.html)

There are several operations that you can perform on a CodeSystem, including:

- [**Subsumption**](http://hl7.org/fhir/codesystem-operation-subsumes.html): This operation allows you to check whether one code in the CodeSystem is a more specific version of another code. For example, you could use it to check whether the code for "fever" is a more specific version of the code for "illness".
- [**Validate code**](http://hl7.org/fhir/codesystem-operation-validate-code.html): This operation allows you to check whether a given code is part of the CodeSystem. You can use it to ensure that the codes you're using in your analysis are valid.
- [**Lookup code**](http://hl7.org/fhir/codesystem-operation-lookup.html): This operation allows you to look up the details of a code in the CodeSystem, such as its description and its relationships to other codes.

#### ValueSet

A ValueSet is a collection of codes from one or more CodeSystems that are used to represent a particular set of concepts. For example, you could create a ValueSet that contains codes for all the diagnostic codes in SNOMED-CT.

There are several operations that you can perform on a ValueSet, including:

- [**Expand**](http://hl7.org/fhir/valueset-operation-expand.html): This operation allows you to expand a ValueSet to include all the codes that are part of the set, along with their descriptions and relationships to other codes. This can be useful if you want to see the full details of the codes in the ValueSet.
- [**Validate Code**](http://hl7.org/fhir/valueset-operation-validate-code.html): This operation is similar to the _validate code_ operation on a CodeSystem, but it allows you to check whether a code is part of a particular ValueSet, rather than just the CodeSystem it belongs to. This can be useful if you want to ensure that the codes you're using in your analysis are part of the specific ValueSet you're working with.

In summary, the Coding, CodeableConcept, CodeSystem, and ValueSet data structures are some of the most important aspects of FHIR Terminology. Understanding these data structures and the operations you can perform on them will be crucial when integrating FHIR Terminology into your Python workflow.

---

⬇️ TO BE COMPLETED

## Parsing data using Pydantic

First we need a convenient way to parse FHIR/JSON into validated typed Python objects. A great package to parse JSON with predefined data models in Python is Pydantic. Even better, somebody has already written the models for FHIR STU3 and FHIR R4. We only need to `pip install fhir.resources` and we can start importing FHIR/JSON into our Python code.

No when fetching ValueSet resource from a FHIR server, we can immediatly parse the response into a validated, typed Python object:

```python
import requests
from fhir.resources.valueset import ValueSet

response = requests.get("http://tx.fhir.org/r4/ValueSet/adverse-event-outcome")
vs_adverse_event_outcome = ValueSet.parse_obj(response.json())
print("Received ValueSet with name: ", vs_adverse_event_outcome.name)
```

## Operations as operators

Next we want to perform terminology operations on these resources without having to perform manual HTTP Requests to the server. We rather want to leverage operator overloading and make the operations on ValueSets and CodeSystems more Pythonic.

### FHIR Terminology server context

When working with Python operators, we specify the operations directly without specifying the FHIR Server. In order to indicate the FHIR Terminology server that needs to execute the operations, we will leverage Python context managers:

```python
with FHIRTerminologyClient("<base-url-of-fhir-server>") as client:
  # ... here come the operations we want to execute
```

### Subsumption

The `$subsumes` api accepts two codes (code A and code B) and returns a relationship code indicating if there is a sumsumtion relationship, equivalence or both codes are completely disjoint. For a Python point of view, this looks like a simple operation between two objects.

We can turn this in the following Python code:

```python
# indicate the terminology server that should be used to execute the operations
with FHIRTerminologyClient("<base-url-to-fhir-terminology>") as fhir_tx_client:
  chronic_fever == new SCTCoding("704425001 |Chronic fever|")
  fever = SCTCoding("704425001 |Fever|")
  assert chronic_fever < ferver, "Expected 'Chronic fever' to be a child of 'Fever'"

```

Note we introduced a little twist to the `$subsumes` API in our Python translation. We immediatly test for a specific operation which results in a boolean value. This comes down to comparing the relationship in the response of the `$subsumes` API to the presumed relationship that is implied by the Python-operator.

The following Python-operators could be a good match for a Pythonic relation test between codes.

| `$subsumes`-outcome | matching Python-operator | meaning                            |
| ------------------- | ------------------------ | ---------------------------------- |
| `equivalent`        | `==`                     | code "A" is equivelnt to code "B"  |
| `subsumes`          | `>`                      | code "A" subsumes code "B"         |
| `subsumed-by`       | `<`                      | code "A" is subsumed by code "B"   |
| `not-subsumed`      | `!=`                     | code "A" and code "B" are disjoint |

### Validate-code

Checking if a code is part of a CodeSystem using the `$validate-code` API does remind a lot to checking if an item is in a list. The natural equivalent in Python is the `in` operator. An example of this API in Python would be:

```python
# indicate the terminology server that should be used to execute the operations
with FHIRTerminologyClient("<base-url-to-fhir-terminology>") as fhir_tx_client:
  sct = CodeSystem.from_canonical("http://snomed.info/sct", server=fhir_tx_server)
  chronic_fever == new SCTCoding("704425001 |Chronic fever|")
  assert chronic_fever in sct, "Expected 'Chronic fever' to be a valid SNOMED-CT concept."
```

### Lookup

A lookup is like a dictionary lookup

> TBC
