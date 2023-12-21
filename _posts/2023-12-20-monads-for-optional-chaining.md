---
title: "Traversing optional FHIR Elements in Python: Making the Complex Simple"
layout: post
author: "Axel Vanraes"
categories:
  - "FHIR"
  - "Python"
---

[FHIR (Fast Healthcare Interoperability Resources)][0] adheres to the 80/20 rule, focusing on providing 80% of the necessary data 20% of the time. While this design philosophy streamlines common use cases, working with FHIR resources becomes challenging due to the optional nature of most elements. Handling optional fields gracefully in Python, especially when nested, can make the code less readable.

At [Tiro.health][2], we leverage [Pydantic models][3] to work with FHIR resources. Pydantic simplifies parsing and validating FHIR/JSON resources, but managing optional fields remains cumbersome. For instance, retrieving a patient's name involves multiple nested checks, leading to code like this:

```python
patient = Patient.parse_file("FHIR-Patient-123.json") # Parse a FHIR Patient from a JSON file
if patient.name is not None:
    if patient.name[0].given is not None:
        if len(patient.name[0].given) > 0:
            print(" ".join(patient.name[0].given[0].value))
        elif patient.name[0].text is not None:
            print(patient.name[0].text)
        else:
            print("No name")
```

To address this, we introduce the Maybe monad, a design pattern commonly used in functional programming languages. While Python lacks native optional chaining, the Maybe monad enhances code readability and reduces verbosity.

_A good YouTube video on monads: [What the Heck Are Monads?!
](https://www.youtube.com/watch?v=Q0aVbqim5pE)_

## Introducing the Maybe Monad

The Maybe monad represents optional values, serving as a container that can hold a value or nothing. Unlike existing Python libraries like [returns][5], our implementation caters to our unique use case which is working with FHIR resources.
i

```python
class Maybe:
    def __init__(self, value):
        """Encapsulate a value."""
        self.value = value

    def __getattr__(self, name):
        """Access nested fields."""
        if self.value is None:
            return self
        # return a None Maybe if the attribute is not found
        return Maybe(getattr(self.value, name, None))

    def __getitem__(self, key):
        """Access list items."""
        if self.value is None:
            return self
        try:
            return Maybe(self.value[key])
        except IndexError:
            return Maybe(None)
```

**Important Notes**:

- Use `getattr` with a default value of `None` to avoid `AttributeError` exceptions.
- In the `__getitem__` method, catch `IndexError` exceptions to return `None` if the index is out of range.

Now, we can simplify nested field access without explicit checks:

```python
print(Maybe(patient).name[0].given[0].value)
```

### Enhancements to the Maybe Monad

To further streamline usage, we introduce additional methods:

#### The `apply` Method

```python
class Maybe:
    # ...
    def apply(self, func):
        if self.value is None:
            return self
        return Maybe(func(self.value))
```

This allows concise application of functions to nested values:

```python
print(Maybe(patient).name[0].given[0].apply(" ".join).value)
```

#### The `__or__` Operator

```python
class Maybe:
    # ...
    def __or__(self, other):
        if self.value is None:
            return other
        return self
```

Now we can chain multiple fields and provide a default value if none are found:

```python
patient = Patient.parse_file("FHIR-Patient-123.json") # Parse a FHIR Patient from a JSON file
maybe_patient = Maybe(patient)
print((
    maybe_patient.name[0].given[0].apply(" ".join) or
    maybe_patient.name[0].text or
    Maybe("No name")
    ).value)
```

**🎉 Et voilà!** We've successfully removed the if/else statements.

There is one additional enhancement we can make to the Maybe monad to make it more useful in other contexts.

#### The `__iter__` Method

```python
class Maybe:
    # ...
    def __iter__(self):
        if self.value is not None:
            try:
                yield from self.value
            except TypeError:
                yield self.value
```

Easily iterate over fields with cardinality `0..*`:

```python
for given in Maybe(patient).name[0].given:
    print(given.value)
```

### Parallels with FHIRPath

Drawing parallels with [FHIRPath][8], a powerful expression language for traversing FHIR resources, the Maybe monad shares similarities:

1. Both use an _empty collection_ or `None` to represent absence of values.
2. Facilitate fluent access and application of functions to nested fields without explicit checks.
3. Allow iteration over fields with cardinality `0..*` without checking for presence.

While FHIRPath offers more advanced features, combining the Maybe monad with FHIRPath might unlock additional power.

### Conclusion

The Maybe monad effectively removes verbosity related to optional fields, making code more readable. However, its implicit behavior requires careful usage to avoid hiding potential bugs. Consider this pattern judiciously, keeping in mind the trade-offs.

[0]: https://www.hl7.org/fhir/ "Fast Healthcare Interoperability Resources"
[2]: https://tiro.health "Tiro.health"
[3]: https://pydantic.dev "Pydantic"
[5]: https://returns.readthedocs.io/en/latest/index.html "returns"
[8]: https://hl7.org/fhirpath/ "FHIRPath"
[0]: https://www.hl7.org/fhir/ "Fast Healthcare Interoperability Resources"
[1]: https://hl7.org/fhir/overview-arch.html#principles "FHIR Principles"
[2]: https://tiro.health "Tiro.health"
[3]: https://pydantic.dev "Pydantic"
[4]: https://james-iry.blogspot.com/2009/05/brief-incomplete-and-mostly-wrong.html "A Brief, Incomplete, and Mostly Wrong History of Programming Languages"
[5]: https://returns.readthedocs.io/en/latest/index.html "returns"
[6]: https://james-iry.blogspot.com/2009/05/brief-incomplete-and-mostly-wrong.html "A Brief, Incomplete, and Mostly Wrong History of Programming Languages"
[7]: https://hl7.org/fhir/R4B/json.html#xml
[8]: https://hl7.org/fhirpath/ "FHIRPath"
[9]: https://en.wikipedia.org/wiki/Monad_(functional_programming) "Monad (functional programming)"
