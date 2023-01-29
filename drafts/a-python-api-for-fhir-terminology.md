---
title: ""
layout: post
permalink: /drafts/fhir-tx-python
---

## Using FHIR Terminology in your Python workflow

### 1. The problem

[FHIR Terminology](https://hl7.org/fhir/terminology-module.html) is a powerful tool for healthcare innovators and software developers, but it can be difficult to integrate into a Python data analysis workflow. One issue is that FHIR Terminology is a REST API, which means that using it in a Python script often involves making a lot of HTTP calls using a library like [requests](https://requests.readthedocs.io). These calls can add technical details to your code that don't contribute to your analysis, and can make your code harder to read.

Another problem is that [FHIR Resources are transferred in JSON format](https://hl7.org/fhir/json.html) (or XML), which means that in Python, you need to decode them into dictionaries. While dictionaries are useful for small data structures, FHIR Resources can have deeply nested structures, and working with dictionaries can make it hard to access the data you need.

Let's say we want to validate that a SNOMED-CT term is in valueset.
You need at almost xx lines of code to do this:

```python
import requests
from urllib.parse import urlencode

def get_name_and_value(param: dict):
  """Extract the name and value[x] of a parameter Backbone element."""
  return param.pop("name"),param.popitem()[1]

coding = {"display": "Fever", "code": "", "system": "http://snomed.info/sct"}
query = urlencode(coding)
response = requests.get(f"{BASE_URL}/ValueSet/xxx/$validate-code?{query}")
assert response.status = 200, f"The call to the FHIR server failed, recieived {response.text}"
# What if the response is an OperationOutcome ? 😞
result_dict = {} 
for param in response.json()["parameters"] # parameters is a list of elements so we need to iterate over all of them 🤔
  name, value = get_name_and_value(param) # 😵‍💫 some anoying parsing logic because of the nested nature of the Parameters resource and FHIR ChoiceTypes
  result_dict.update({name: value})

print("Is Fever part of ValueSet xxx? " + str(result_dict["result"]]))

```

Note that it is almost impossible to write this example without the FHIR docs side-by-side 😅. The problem is that we need to take care of things like URL-construction, making HTTP requests and finally parsing and validation of responses.
Fortunatly, we can make this easier by abstracting away technical details that are relevant from a FHIR perspective.

#### The ingredients

The first thing to notice is that we handle all our HTTP requests and responses manually. We'd rather work with an SDK that helps us constructing the relevant URL's, make calls, and check the responses. Some people in the FHIR community have already worked on this and created `fhir-py`. It's essentially a client that takes care of building the right URL's and then excute CRUD operations or other resource-specific operations on a FHIR server.

Next we need to parse and validate the response. [`fhir-py`](https://github.com/beda-software/fhir-py) is FHIR-version independent and leaves us with a JSON-decoded resource in the response. This means that we still need to traverse the dictionary we've received without having validation or type-annotations. Another open-source effort that can help us here is [`fhir.resources`](https://github.com/nazrulworld/fhir.resources). The libary is built on top of [`pydantic`](https://docs.pydantic.dev/), the goto library for model based JSON parsing and validation. Nazrul has built models for FHIR DSTU2, STU3 and R4 (including R4B) that help use parsing JSON, validating it's structure agains the models and returning typed-objects. 

Finally, we've still have to tackle redundancies due to way FHIR is architected. FHIR is built for interoperability and not for consice Python-code. An example of this, when working with FHIR Terminology, is the [**Parameters**](http://hl7.org/FHIR/parameters.html) resources that is used for defining in- and outgoing parameters in [extend operations](https://hl7.org/fhir/operations.html). The structure of Parameters is extremeley redundant from a Python perspective, we're we typically use `func(**kwargs)` if we don't know the structure of parameters beforehand. 

On top of that, we can map some of these [extended operations]() in the Terminology module to some commonly used Python-operators using [operator overloading](https://realpython.com/operator-function-overloading/) and [magic-methods](https://www.geeksforgeeks.org/dunder-magic-methods-python/). This will not only remove boilerplate code, but make our code more Pythonic and readable 😍. 

In the rest of this blogpost we will go through the process of creating a micro library that helps us interacting with FHIR Terminology resources while solving the issues discussed above and aiming for the following objectives:

- ✅ easy access to terinology resources
- ✅ maximal reuse of existing open-source efforts
- ✅ Pythonic primimtives


### 2. Key data structures and operations in FHIR Terminology

> The following sections are pure review of resources and operations defined in the FHIR Terminology spec.
> Feel free to skip this section and [go to next](#3-building-our-terminology-client) if you are familiar with this.

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

#### ConceptMap


---

### 3. Building our Terminology Client

With tiny bit of code we can build a client on top of `fhirpy` and `fhir.resources` that handles operations on terminology resources and abstracts details about HTTP requests, parsing and validation. Let's focus on the `$expand` and `$validate-code` operation on ValueSets. The process is similar for the other operations and you can browse through the final source code [here ⚠️]()

#### Accessing FHIR Resources using `fhirpy`

CRUD operations using `fhirpy` is simple, the follow snippet speaks for itself:

```python
from fhirpy import SyncFHIRClient

 # define our client pointing to our server base-url
client = SyncFHIRClient("http://tx.fhir.org/r4/")

# fetch a resource of type ValueSet with id="adverse-event-outcome"
vs = client.resource("ValueSet", id="adverse-event-outcome")
print(vs.to_resource())
```

The ease of use of `fhirpy` is also the problem, it doesn't help use further. `fhirpy` takes care of building the URL, handling HTTP-codes and OperationOutcome's but there is no special support for the numerous extended operations like `$expand`, `$validate-code$`, `$subsumes`, ...

Fortunatly, the authors have made the API very extendable and we can easily extend the libary to support comon terminology operations.

#### Extending the SyncClient for terminnology operations:
Let's build a special `SyncFHIRTerminology` that covers the `$validate-code` operation for ValueSets:

```python
from abc import ABC
from fhirpy import SyncFHIRClient
from fhirpy.lib import SyncFHIRResource
from .util import dict_to_params, params_to_dict

class SyncValueSet(SyncFHIRResource, ABC):
    """ValueSet Client which can handle $expand and $validate-code """
    def expand(self):
        result = self.execute(
            "$expand",
            method="POST",
        )
        return result

    def validate_code(self, coding):
        result = self.execute(
            "$validate-code",
            method="POST",
            data={
              "resourceType": "Paramters",
              "parameter":[
                {"name": "coding", "valueCoding": coding}
              ]
            }
        )
        return params_to_dict(result) # convert the Parameters to a simple dict {name: value}

class SyncFHIRTerminologyClient(SyncFHIRClient):
    def resource(self, resource_type=None, **kwargs):
        if resource_type == "ValueSet":
            return SyncValueSet(self, resource_type=resource_type, **kwargs)
        return super().resource(resource_type, **kwargs)
```

The script above, allows us to simplify our first example drastically:

```python
from fhir_tx_client import SyncFHIRTerminologyClient
client = SyncFHIRClient("http://tx.fhir.org/r4/")
result_dict = client.validate_code(coding={"display": "Fever", "code": "", "system": "http://snomed.info/sct"})

print("Is Fever part of ValueSet xxx? " + str(result_dict["result"]]))
```

Implementing a new operation in `fhirpy` is quite simple. The only awkwardness remainiong is transforming our Python `**kwargs` (the dictionary of arguments in a function) to a Parameters-resource and transforming back the result to something usefull in Python. 

#### Building and parsing FHIR/JSON resources using `fhir.resources`

*TBC: describe usage of Pydantic*


#### Operations as operators

Our final step in building this library, is to make use of commonly used Python operators to reduce the boilerplate even more. 

The key insight here is that ValueSet's behave like a list of `coding` elements and `ConceptMaps` behave like mappings or dictionaries with `coding` elements both as key and as value. This naturally raises the question if we can make use of typical Python syntax that we're used to work with.

For ValueSets we would expect something like this to work:

```python
vs = FHIRTerminologyClient("http://tx.fhir.org/r4") \
      .resource("ValueSet", id="http://snomed.info/sct?fhir_vs=isa/")

code1 = Coding(display="Chronic fever", code="704425001", system="http://snomed.info/sct")

msg = "Is 'Chonic fever' in the ValueSet containing all descendants of 'Fever?'"
print(msg, code1 in vs)

```

---

*TBC*

#### CodeSystem -- `$subsumes`

The `$subsumes` api accepts two codes (code A and code B) and returns a relationship code indicating if there is a sumsumtion relationship, equivalence or both codes are completely disjoint. A logical Pythonic way to express this operation is a simple boolen expression between two objects.

We can turn subsumption in the following Python code:

```python
  chronic_fever == new SCTCoding.from_sct("704425001 |Chronic fever|")
  fever = new SCTCoding.from_sct("704425001 |Fever|")

  assert chronic_fever < ferver, "Expected 'Chronic fever' to be a child of 'Fever'"

```
### Final thoughts

The whole library 

 *TBC* 

### Adendum: mapping of Python-operations to FHIR Terminology operations



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

#### CodeSystem -- `$lookup`

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

#### ValueSet -- `$validate-code`

This operation can be implemented identically like the `validate-code` operation on a CodeSystem.

```python
  fhir_tx = FHIRTerminologyClient("http://tx.fhir/torg/r4")
  vs_adverse_event_outcome = fhir_tx.ValueSet["adverse-event-outcome"] # read the ValueSet given it's id

  code = SCTCoding.from_sct("398056004 |Transient abnormality with full recovery|")
  assert code in vs_adverse_event_outcome, "Expected 'Transient abnormality with full recovery' to be part of ValueSet 'AdverseEventOutcome'.

```

#### ValueSet -- `$expand`

Expansion of valueset is a very specific operation which has no obvious Python equivalent at first site. Instead of translating the operation in a Python operator, it makes more sense to create a dedicated `expand()` function for ValueSet expansion. The resulting code interface in Python ends up like this:

```python
fhir_tx = FHIRTerminologyClient("http://tx.fhir/torg/r4")
vs_adverse_event_outcome = fhir_tx.ValueSet["adverse-event-outcome"] # read the ValueSet given it's id

assert vs_adverse_event_outcome.expand().expansion is not None, "Expected an expanded valueset."
```

### FHIR Terminology server context

An attentive reader may have noticed that there is something missing in the Python snippets above. We translated the REST operations to simple readable Python expressions. But in these Python expression we do not specify the FHIR server where those operations must be executed. Take the case of the `$subsumes`-operation: we implement the `__le__()` function, but how will we now inside that function which FHIR-server to call?

To solve this issues without having to specify the FHIR server for each operation, we'll leverage context-managers.
// When working with Python operators, we specify the operations directly without specifying the FHIR Server. In order to indicate the FHIR Terminology server that needs to execute the operations, we will leverage Python context managers:

```python
with SyncFHIRTerminologyClient("<base-url-of-fhir-server>") as fhir_tx:
  # ... here come the operations we want to execute
```
