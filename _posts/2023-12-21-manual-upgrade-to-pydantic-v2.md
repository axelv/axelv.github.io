---
title: "Migrating to Pydantic v2: A Journey with FHIR Models"
author: "Axel Vanraes"
categories:
  - "FHIR"
  - "Python"
  - "Pydantic"
---

When dealing with [FHIR][1] in Python, having typed models to represent resources is essential. FHIR resources are nested structures with field names (officially FHIR Elements) that aren't always consistent across resource types. Handling them with dictionaries can be tricky and error-prone. At [Tiro.health][2], we leverage [Pydantic models][3] to streamline the process of working with FHIR resources. Pydantic simplifies parsing and validating FHIR/JSON resources and provides valuable type hints for IDEs.

Recently, Pydantic released [version 2][4], introducing significant improvements. The library's internals were rewritten in Rust, enhancing speed and memory efficiency. Observing validation as a bottleneck in our backend services, I decided to explore the migration of our models to Pydantic v2.

## How do we generate our models?

In 2021, during our company launch, we initially crafted models manually. However, as we expanded our work with more resources, we realized the need for a better solution. After exploring existing libraries and facing issues with types in [`fhir.resources`][5], we discovered [`fhir-py-types`][6] from [beda.software](https://beda.software/). This library generates Pydantic models from the FHIR specification, providing a single file with all models. An open-source example of such a generated file can be found [here](https://github.com/Tiro-health/FHIRkit/blob/v1.0/fhirkit/r5.py)

The challenge now is migrating this file to Pydantic v2.

## Migrating to Pydantic v2 by hand

Following Samuel Colvin and team's migration guide, I made the following changes:

1. Replaced `update_forward_ref()` with `model_rebuild()` for all resources:

   ```diff
   -Patient.update_forward_refs()
   +Patient.model_rebuild()
   ```

2. Moved model config from Metaclass arguments to the `model_config` field:

   ```diff
   -class Patient(BaseModel, extra=Extra.forbid, validate_assignment=True):
   +class Patient(BaseModel):
   +    model_config = ConfigDict(extra="forbid", validate_assignment=True)
   ```

Unfortunately, it wasn't that simple...

### Python's maximum recursion depth

Due to the large number of models (700+), we hit Python's maximum recursion depth.
This can easily be tested by simply running the module containing models. This results in a **RecursionError**:

```bash
Traceback (most recent call last):
  File "/Users/axelvanraes/dev/fhirkit/./fhirkit/r4.py", line 20916, in <module>
    Account.model_rebuild()
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/main.py", line 470, in model_rebuild
    return _model_construction.complete_model_class(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/_internal/_model_construction.py", line 491, in complete_model_class
    schema = cls.__get_pydantic_core_schema__(cls, handler)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/main.py", line 578, in __get_pydantic_core_schema__
    return __handler(__source)
//... snip ...
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/_internal/_generate_schema.py", line 810, in match_type
    return self._match_generic_type(obj, origin)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/_internal/_generate_schema.py", line 829, in _match_generic_type
    from_property = self._generate_schema_from_property(origin, obj)
                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/_internal/_generate_schema.py", line 592, in _generate_schema_from_property
    with self.defs.get_schema_or_ref(obj) as (_, maybe_schema):
  File "/opt/homebrew/Cellar/python@3.11/3.11.6_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/contextlib.py", line 137, in __enter__
    return next(self.gen)
           ^^^^^^^^^^^^^^
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/_internal/_generate_schema.py", line 2083, in get_schema_or_ref
    ref = get_type_ref(tp)
          ^^^^^^^^^^^^^^^^
  File "/Users/axelvanraes/dev/fhirkit/.venv/lib/python3.11/site-packages/pydantic/_internal/_core_utils.py", line 93, in get_type_ref
    origin = get_origin(type_) or type_
             ^^^^^^^^^^^^^^^^^
  File "/opt/homebrew/Cellar/python@3.11/3.11.6_1/Frameworks/Python.framework/Versions/3.11/lib/python3.11/typing.py", line 2431, in get_origin
    if isinstance(tp, (_BaseGenericAlias, GenericAlias,
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
RecursionError: maximum recursion depth exceeded in __instancecheck__
```

It's hard to pinpoint the exact cause of this error. However, it seems to be related to the `AnyResource` type. This type is used to represent any FHIR resource. It's used in the `contained` of every resource and hence is the source of the recursion.
Manually reducing the number of models in `AnyResource` to a subset of resources, makes the error disappear. This is a good indication that the number of models is the cause of the error.

After some trial and error, we came up with the following solution:

```diff
+ from typing import Any
class Patient:
+    contained: Optional_[List_["Any"]] = None
-    contained: Optional_[List_["AnyResource"]] = None
```

This change, though breaking, was acceptable for our use case as we always specify contained resources through custom models.

### Reducing model build time

Pydantic v2 takes more time to build models, sacrificing some loading time for drastically improved parsing and validation performance. However, loading times of almost **40 seconds** is unacceptable. I hope Pydantic is built with larger projects in mind and will improve this in the future. At least, we found another workaround to reduce loading times:

1. Parsed extensions and modifier extensions as dicts:

   ```diff
   -    extension: Optional_[List_[Extension]] = None
   +    extension: Optional_[Dict_[str, Any]] = None
   ```

   and

   ```diff
   -    modifierExtension: Optional_[List_[Extension]] = None
   +    modifierExtension: Optional_[Dict_[str, Any]] = None
   ```

   If we use extensions, we always specify them through custom models. So this change was acceptable.

2. Parsed `Bundle.entry.resource` as a dict:

   ```diff
   -    resource: Optional_[AnyResource] = None
   +    resource: Optional_[Dict_[str, Any]] = None
   ```

   This change was acceptable as we always specify the resource type through custom models.

Loading times were reduced to **5 seconds**, a significant improvement. However, it's still not ideal. Since we only use a subset of models in each service, we could split the models into separate files. But this is a topic for another time.

## What's next?

The manual migration revealed pain points that need addressing. Some improvements to the `fhir-py-types` library before migrating include

1. **Splitting models into separate files** to reduce loading and build time, making maintenance more manageable. The assumption here is that each service only needs a subset of models.

2. Specifying **model config in the BaseModel** to avoid repeating the migration for each resource type.

[1]: https://www.hl7.org/fhir/ "Fast Healthcare Interoperability Resources"
[2]: https://tiro.health "Tiro.health"
[3]: https://pydantic.dev "Pydantic"
[4]: https://pydantic-docs.helpmanual.io/usage/v2_upgrade_guide/ "Pydantic v2 upgrade guide"
[5]: https://github.com/nazrulworld/fhir.resources "fhir.resources"
[6]: https://github.com/beda-software/fhir-py-types "fhir-py-types"
