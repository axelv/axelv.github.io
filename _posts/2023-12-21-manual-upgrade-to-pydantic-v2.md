---
title: "Migrate generated FHIR models to Pydantic v2"
author: "Axel Vanraes"
categories:
  - "FHIR"
  - "Python"
  - "Pydantic"
---

When working with [FHIR][1] in Python, you definitly need typed models to represent your resources. FHIR resources are nested structures with field names (FHIR Elements officially) that are not always consistent accross resource types.So working with dictionaries is very tricky and error prone. At [Tiro.health][2], we use [Pydantic models][3] to work with FHIR resources. Pydantic simplifies parsing and validating FHIR/JSON resources and offers type hints for IDEs.

Recently, Pydantic released [version 2][4] with a lot of improvements. They rewrote the internals of the library in Rust, which makes it faster and more memory efficient. We've seen that parsing in our backend services becomes a bottleneck, so I wanted to see what it would take to upgrade our models to Pydantic v2.

## How do we generate our models?

In 2021, when we launched the company, we used to create our models by hand. We had a few models and we were not sure if we would stick with Pydantic. But as we started to work with more resources, we quickly realized that we needed a better solution. We started to look at the existing libraries, found [`fhir.resources`][5] but experienced a [lot of issues with their types](https://github.com/nazrulworld/fhir.resources/issues/60) which is the main feature we were looking for. Last year, we encountered [`fhir-py-types`][6] from [beda.software](https://beda.software/). They took a different approach and built a library that generates Pydantic models from the FHIR specification. We started to use it and it worked great.

The result that `fhir-py-types` generate is a single file with all the models. Here is [an example](https://raw.githubusercontent.com/Tiro-health/FHIRkit/v1.0/fhirkit/r5.py). Now the challenge is to migrate this file to Pydantic v2.

## Migrating to Pydantic v2 by hand

Samuel Colvin and team have made an excellent migration guide. Following their guide I had to make the following changes:

1. Replace `update_forward_ref()` with `model_rebuild()` for all the resources
   ```diff
   -Patient.update_forward_refs()
   +Patient.model_rebuild()
   ```
2. Move model config from the Metaclass arguments to the `model_config` field.
   ```diff
   -class Patient(BaseModel, extra=Extra.forbid, validate_assignment=True):
   +class Patient(BaseModel):
   +    model_config = ConfigDict(extra="forbid", validate_assignment=True)
   ```

### Fixing errors

Initially, I thought that this would be enough to migrate the models 🙈. But when I tried to run the tests, I got a lot of errors.

Just loading the module containing the models failed with the following error:

```bash
python r4.py
```

**Error traceback:**

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

After a lot of try and error I found the following workaround:

```diff
+ from typing import Any
class Patient:
+    contained: Optional_[List_["Any"]] = None
-    contained: Optional_[List_["AnyResource"]] = None
```

> **Note:** This is a breaking change. But in our case, we always specify the contained resources by creating custom models. So we can easily migrate to the new type.

I'm not sure why this works, but it does. Probably, Pydantic v2 is doing more type checking when building the models compared to v1. Give we have **700+ models with recursive references**, it's not unthinkable that we hit Python's recursion limit.

### Reducing model build time.

It is known that Pydantic v2 takes a bit more time to build the models in favor of faster parsing and validation performance. But when I tried to load the models, it took way more time.
After inserting some log lines, I noticed that **it takes almost 40 seconds to load the models ⚠️**. This is a huge regression compared to Pydantic v1 which takes barely 1 second.

Loading times of 40 seconds is of course not acceptable especially since we have services that scale to zero and need to load the models on startup. So I started to look for ways to reduce the loading time. I found the following improvements:

1. Parse extensions as a dict

   ```diff
   -    extension: Optional_[List_[Extension]] = None
   +    extension: Optional_[Dict_[str, Any]] = None
   ```

   and

   ```diff
   -    modifierExtension: Optional_[List_[Extension]] = None
   +    modifierExtension: Optional_[Dict_[str, Any]] = None
   ```

   > **Note:** This is a breaking change. But in our case, we always specify the extensions by creating custom models. So we can easily migrate to the new type.

2. Parse `Bundle.entry.resource` as a dict

   ```diff
   -    resource: Optional_[AnyResource] = None
   +    resource: Optional_[Dict_[str, Any]] = None
   ```

   > **Note:** This is a breaking change. But most of the time we don't use Bundles without knowning the subset of resources we can expect. So we can easily migrate to the new type.

## What's next?

The manual migration definitely did learn where the pain points are. It will be important to refactor the `fhir-py-types` library around these pain points. I'm thinking about the following improvements:

1. Split the models in seperate files. This will reduce the model loading and build time and make it easier to maintain the models. I think it is fairly straightforward to bundle all the FHIR Datatypes in a single file and have a seperate file for each resource type. But what to do with the union type `AnyResource` ?

2. Specify model config in the BaseModel so we don't have to repeat the migration for each resource type.

[1]: https://www.hl7.org/fhir/ "Fast Healthcare Interoperability Resources"
[2]: https://tiro.health "Tiro.health"
[3]: https://pydantic.dev "Pydantic"
[4]: https://pydantic-docs.helpmanual.io/usage/v2_upgrade_guide/ "Pydantic v2 upgrade guide"
[5]: https://github.com/nazrulworld/fhir.resources "fhir.resources"
[6]: https://github.com/beda-software/fhir-py-types "fhir-py-types"
