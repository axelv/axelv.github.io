---
title: "Registries on FHIR: stop reinventing the form"
layout: post
author: "Axel Vanraes"
published: false
categories:
  - "FHIR"
  - "Interoperability"
  - "Registries"
---

![Stop Redoing the Form. Build Belgian Registries on FHIR.](/_posts/registries-on-fhir.png)

In Belgium we run a lot of ad-hoc registries. National ones like the Belgian Cancer Registry and QERMID, but also regional and network-level initiatives like VZNKUL FLIPR. Each one comes with its own data dictionary, its own care pathway, sometimes a set of quality indicators, and almost always a form that needs to land inside the clinician's workflow.

That form is where most of the hidden cost lives.

To make a registry actually work in practice, someone has to translate the data dictionary into a form, fit that form to how clinicians already work, and then wire it into the EHR. Automation can soften the administrative burden, but the form itself stays a prerequisite: it's where clinical validation happens. Today every registry redoes that work from scratch. New schema, new form, new integration, new transmission pipeline, new version management, all quietly absorbed by application managers and data analysts in the hospitals.

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

The simple truth: every new registry doesn't need new infrastructure. It needs a published StructureDefinition, a Questionnaire bound to it, and a hospital that already speaks FHIR.

We have most of the pieces. We just need to stop redoing the form.

*Curious which Belgian registries you'd like to see modeled next. Drop a name in the comments.*

#FHIR #HealthTech #Interoperability #DigitalHealth #StructuredDataCapture #Registries
