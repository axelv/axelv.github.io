# Using FHIR Terminology your Python workflow

## Introduction
- FHIR Terminology has some wonderfull features but as a data scientist it's hard to use it in my analysis workflow.
-  First of all, FHIR Terminology is a REST API and this means lots of [HTTP calls using the `requests` library](https://requests.readthedocs.io/en/latest/). Calls to a REST API involve some technical details that do not contribute to the data analysis we are making. It's only polluting the code readability.
- Secondly, FHIR Resources are tranfered in JSON and in Python we decode this to dictionaries. Dictionaries are usefull for small temporary data structures but FHIR Resources are way to big to put in a dictionary. I don't know by heart which keys their are available in a ValueSet dictionary. Instead, we want to have dedicated class objects with attributes that help us accessing the attributes we need.

## Important aspects FHIR Terminology
Before goinging into the details of how we handle terminology in Python, let's discuss the most important data structures related FHIR Terminology

### Coding
A Coding is a the most basic structure refering to a single code that is part of a CodeSystem. An example of such a 
### CodeableConcept

### CodeSystem

### ValueSet

## Parsing data using Pydantic

