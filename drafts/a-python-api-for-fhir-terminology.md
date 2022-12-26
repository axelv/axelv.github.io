---
title: ""
layout: post
permalink: /drafts/fhir-tx-python
---

## Using FHIR Terminology your Python workflow

[FHIR Terminology](https://hl7.org/fhir/terminology-module.html) is a powerful tool for healthcare innovators and software developers, but it can be difficult to integrate into a Python data analysis workflow. One issue is that FHIR Terminology is a REST API, which means that using it in a Python script often involves making a lot of HTTP calls using a library like [requests](https://requests.readthedocs.io). These calls can add technical details to your code that don't contribute to your analysis, and can make your code harder to read.

Another problem is that [FHIR Resources are transferred in JSON format](https://hl7.org/fhir/json.html) (or XML), which means that in Python, you need to decode them into dictionaries. While dictionaries are useful for small data structures, FHIR Resources can be quite large, and working with dictionaries can make it hard to access the data you need.

To make it easier to use FHIR Terminology in your workflow, you need a way to work with **validated, typed objects** for all the FHIR resources and elements related to terminology. And you want operations on these elements to be easy to perform using **Python operators** or functions.

In this blog post, we'll explore some ways to integrate FHIR Terminology into your Python workflow, and make it easier to use in your data analysis projects.

### Key data structures and operations in FHIR Terminology

> The follow is pure review of resources and operations defined in the FHIR Terminology spec.
> Feel free to skip this section if you are familiar with this.

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

First we need a convenient way to parse FHIR/JSON into validated typed Python objects. A great package to parse JSON with predefined data models in Python is Pydantic. There are a couple of initiatives that have already build such models for FHIR. At [Tiro.health](https://tiro.health) we've been working on [FHIRkit](https://github.com/Tiro-health/fhirkit). But there are other promissing initiatives like [fhir.resources](https://github.com/nazrulworld/fhir.resources).

No when fetching a ValueSet resource from a FHIR server, we can immediatly parse the response into a validated, typed Python object:

```python
import requests
from fhir_tx.valueset import ValueSet

valueset_url = "http://tx.fhir.org/r4/ValueSet/adverse-event-outcome"
response = requests.get(valueset_url)
vs_adverse_event_outcome = ValueSet.parse_obj(response.json())
print("Received ValueSet with name: ", vs_adverse_event_outcome.name)
```

## CRUD operations

In order to make it easier to perform CRUD operations let's abstract this away by implementing a client that implements the `__getitem__` method:
Hiding away

```python
fhir_tx = FHIRTerminologyClient("http://tx.fhir/torg/r4")
vs_adverse_event_outcome = fhir_tx.ValueSet["adverse-event-outcome"] # read the ValueSet given it's id
fhir_tx.ValueSet["adverse-event-outcome"] = vs_adverse_event_outcome # create or update ValueSet
del fhir_tx.ValueSet["adverse-event-outcome"] # remove ValueSet
```

The above abstractions allows to reuse the same fetch logic and base URL easily. It also creates place where you can put more advance validation and exception handling logic.

## Operations as operators

Next we want to perform terminology operations on these resources without having to build HTTP Requests manually. Moreover, it becomes tedious if we need to specify FHIR server for each operation in script where we want to perform a whole sequence of operations. In the next sections we will leverage [OOP](https://en.wikipedia.org/wiki/Object-oriented_programming) and operator overloading in Python and make the operations on ValueSets and CodeSystems more efficiant (in lines of code), readable and Pythonic.

Let's go through each FHIR operation and formulate a matching Pythonic equivalent.

### CodeSystem -- `$subsumes`

The `$subsumes` api accepts two codes (code A and code B) and returns a relationship code indicating if there is a sumsumtion relationship, equivalence or both codes are completely disjoint. A logical Pythonic way to express this operation is a simple boolen expression between two objects.

We can turn subsumption in the following Python code:

```python
  chronic_fever == new SCTCoding.from_sct("704425001 |Chronic fever|")
  fever = new SCTCoding.from_sct("704425001 |Fever|")

  assert chronic_fever < ferver, "Expected 'Chronic fever' to be a child of 'Fever'"

```

There are several things to note here:

1. We implemented a Pydantic model for codings and added some convenience methods that parse a SNOMED-CT notation string.
2. We've introduced a little twist to the `$subsumes` API in our Python translation. Instead of returning the hierarchical relation, we immediatly test for a specific relation and return a boolean value. This comes down to comparing the relationship in the response of the `$subsumes` API to the presumed relationship that is implied by the Python-operator.

The following Python-operators could be a good match for a Pythonic relation test between codes.

| `$subsumes`-outcome | matching Python-operator | meaning                            |
| ------------------- | ------------------------ | ---------------------------------- |
| `equivalent`        | `==`                     | code "A" is equivalent to code "B" |
| `subsumes`          | `>`                      | code "A" subsumes code "B"         |
| `subsumed-by`       | `<`                      | code "A" is subsumed by code "B"   |
| `not-subsumed`      | `!=`                     | code "A" and code "B" are disjoint |

### CodeSystem -- `$validate-code`

Checking if a code is part of a CodeSystem using the `$validate-code` API does remind a lot to checking if an item is in a list. This makes the `in` operator the natural equivalent to implement this in Python. An example of that would look like in code:

```python
  fhir_tx = FHIRTerminologyClient("http://tx.fhir/torg/r4")
  sct = fhir_tx.CodeSystem.get(uri="http://snomed.info/sct")

  chronic_fever == SCTCoding.from_sct("704425001 |Chronic fever|")

  assert chronic_fever in sct, "Expected 'Chronic fever' to be a valid SNOMED-CT concept."
```

### CodeSystem -- `$lookup`

A lookup operation does immediatly remind of dictionaries in Python. Using the `__getitem__` operator we can mimic dictionary syntax on our on objects.
This would result in the followin code:

```python
  # create an instance of codesystem
  fhir_tx = FHIRTerminologyClient("http://tx.fhir/torg/r4")
  sct = fhir_tx.CodeSystem.get(uri="http://snomed.info/sct")
  chronic_fever == new SCTCoding(code=704425001)

  result = sct[chronic_fever]
  result.display === "Chronic fever"
```

### ValueSet -- `$validate-code`

This operation can be implemented identically like the `validate-code` operation on a CodeSystem.

```python
  fhir_tx = FHIRTerminologyClient("http://tx.fhir/torg/r4")
  vs_adverse_event_outcome = fhir_tx.ValueSet["adverse-event-outcome"] # read the ValueSet given it's id

  code = SCTCoding.from_sct("398056004 |Transient abnormality with full recovery|")
  assert code in vs_adverse_event_outcome, "Expected 'Transient abnormality with full recovery' to be part of ValueSet 'AdverseEventOutcome'.

```

### ValueSet -- `$expand`

Expansion of valueset is a very specific operation which has no obvious Python equivalent at first site. Instead of translating the operation in a Python operator, it makes more sense to create a dedicated `expand()` function for ValueSet expansion. The resulting code interface in Python ends up like this:

```python
fhir_tx = FHIRTerminologyClient("http://tx.fhir/torg/r4")
vs_adverse_event_outcome = fhir_tx.ValueSet["adverse-event-outcome"] # read the ValueSet given it's id

assert vs_adverse_event_outcome.expand().expansion is not None, "Expected an expanded valueset."
```

### FHIR Terminology server context

An attentive may have noticed that there is something missing in the Python snippets above. We translated the REST operations to simple readable Python expressions. But in these Python expression we do not specify the FHIR server where those operations must be executed. Take the case of the `$subsumes`-operation: we implement the `__le__()` function, but how will we now inside that function which FHIR-server to call?

To solve this issues without having to specify the FHIR server for each operation, we'll leverage context-managers.
// When working with Python operators, we specify the operations directly without specifying the FHIR Server. In order to indicate the FHIR Terminology server that needs to execute the operations, we will leverage Python context managers:

```python
with FHIRTerminologyClient("<base-url-of-fhir-server>") as fhir_tx:
  # ... here come the operations we want to execute
```

## Final before/after example:

Let's now look how this Python client has simplified our code for an reallife terminology usecase:

## Conclusion
