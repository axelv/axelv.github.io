---
title: "Registries on FHIR: stop reinventing the form"
layout: post
author: "Axel Vanraes"
categories:
  - "FHIR"
  - "Interoperability"
  - "Registries"
---

![Stop Redoing the Form. Build Belgian Registries on FHIR.](/assets/registries-on-fhir-wallpaper.png)

In Belgium we run a lot of ad-hoc registries. National ones like the Belgian Cancer Registry and QERMID, and a long tail of regional and network-level initiatives. Each one comes with its own data dictionary, its own care pathway, sometimes a set of quality indicators and a moment of clinical validation that today almost always shows up as a standalone form inside the clinician's workflow.

That form is where most of the hidden cost lives.

Most of the data a registry asks for already lives in the EHR. The reason teams keep building standalone forms isn't that the data is missing, it's that today's EHRs make it painful to find it, export it, and bind it to a registry's schema. So the work gets pushed to the clinician: a new form, duplicate registration, and a transmission pipeline owned by hospital data analysts.

But even when the data *is* there, someone still has to validate it clinically and add the context-specific bits a chart review can't infer. So the form-shaped surface doesn't disappear, it just shrinks. The question is whether every registry rebuilds that surface from scratch, or whether hospitals can build it once on top of infrastructure they already have.

And this is happening exactly when hospitals are investing heavily in FHIR.

FHIR is well known as the *language* of healthcare interoperability, and legislators around the world are now enforcing it. But what often gets missed is that FHIR also ships with a powerful set of *infrastructure primitives*. Two of them map almost perfectly onto what a registry needs.

### 1️⃣ FHIR StructureDefinitions: your data dictionary, in a standard

StructureDefinitions are the base layer on which all FHIR resources are built. They describe a data structure, allow validation of incoming data, and feed the code-generation tooling developers already use.

At its core, a StructureDefinition is a list of data elements, which is exactly what a registry data dictionary is. Define it once, publish it as a FHIR Implementation Guide, and you immediately get shareable, human and machine readable documentation. The same artifact then drives validation in any FHIR-compliant tool.

### 2️⃣ FHIR Structured Data Capture: forms as first-class citizens

FHIR SDC is a whole best-practice guide (and an active community of tooling) dedicated to forms. It standardizes form definitions and explains how the captured data can be extracted for secondary use.

The neat trick: every field in a FHIR Questionnaire can be tagged with the data element it represents. That clean separation between *data dictionary* (StructureDefinition) and *form definition* (Questionnaire) means forms can be adapted to a clinician's workflow (different EHR, different specialty, different language) while staying compatible with the underlying registry schema.

### What this looks like in practice

To make it concrete, I built two example implementation guides:

- **QERMID register logical models** → [https://axelv.github.io/qermid/](https://axelv.github.io/qermid/). The RIZIV/INAMI registries (Orthopride, Pacemakers, ANGIO, TAVI), 23 Data Collection Definitions modeled as FHIR Logical Models.
- **Belgian Cancer Registry on FHIR** → [https://axelv.github.io/bcr/](https://axelv.github.io/bcr/). The BCR forms, the *Cancer in Belgium* research dataset, and the cyto-histopathology screening supplements, modeled directly from the public source documents.

Both are early drafts, but they already show the same three benefits:

1. **Human and machine readable data dictionaries** out of the box.
2. **Reuse of existing data capture infrastructure** that hospitals are already investing in.
3. **Seamless exchange of form definitions**, so the form follows the schema, not the other way around.
4. **Pre-population from existing FHIR data**, turning the form from a duplicate-entry chore into a thin validation-and-augmentation layer over data the EHR already holds.

Conclusion: every new registry doesn't need a new form built from zero. It needs a published StructureDefinition, a Questionnaire bound to it, and an EHR that can serve the data it already holds.

We have most of the pieces. We just need to stop redoing the form.

*Curious which Belgian registries you'd like to see modeled next. Drop a name in the comments.*

#FHIR #HealthTech #Interoperability #DigitalHealth #StructuredDataCapture #Registries
