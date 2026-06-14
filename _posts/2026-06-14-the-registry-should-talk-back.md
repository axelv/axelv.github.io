---
title: "The registry should talk back"
layout: post
author: "Axel Vanraes"
categories:
  - "FHIR"
  - "Interoperability"
  - "Registries"
---

![A clinician submits a registration over FHIR; the registry validates and the answer flows back along the same wire.](/assets/registry-talks-back.png)

Belgium runs quite some registries: the Belgian Cancer Registry, QERMID, and a long tail of regional and network-level initiatives. Each one shows up with its own data dictionary, its own form, its own integration, and its own transmission pipeline. And each one gets rebuilt from scratch, quietly absorbed by application managers and data analysts inside the hospitals.

I've argued before that we should [stop reinventing the form](https://axelv.github.io/fhir/healthtech/interoperability/registries/structureddatacapture/2026/05/16/registries-on-fhir.html), that FHIR already gives us the primitives to publish a registry's data dictionary once and bind forms to it. That's still true. But after talking to the registry experts from the Belgian Cancer Registry, the FHIR-A-THON taught us the form is only half of the story.

## The form was only half of the story

Most of what a registry asks for already lives in the EHR or the lab system, so the form should arrive mostly pre-filled and let the clinician do the last mile; that was the argument [in the previous post](https://axelv.github.io/fhir/healthtech/interoperability/registries/structureddatacapture/2026/05/16/registries-on-fhir.html), and FHIR SDC's `$populate` already does it.

What we hadn't realized is that a perfectly pre-filled form is still a one-way data flow for the registry. You can make submitting effortless and *still* never know whether the registration quality was any good.

## The registry has to answer back

Here's the part nobody had really built. Validation rules in a cancer registry are not trivial. They're cross-field, they encode oncology business logic, and they often can't run at the point of care. Which means the verdict — *is this registration actually acceptable?* — comes back **later**.

Today, that "later" mostly doesn't come back at all. You submit a registration and hope it's ok. If something's wrong, the clinician who could fix it has long since moved on, and the correction loop runs over email, phone, or not at all.

A registration shouldn't be a *submission*. It should be a *conversation*: the hospital sends what it has, the registry validates in the background, and the result: accepted, or "please fix the topography field"; comes back to someone who can act on it. And a conversation needs an active server, not a mailbox: a document channel like eHealthBox can deliver a form but can't return a result or hold the state of a workflow that spans several attempts, so the registry has to run a FHIR server the hospital can talk to like any other API.

## What we actually built: a worklist

The shape that fell out of this is a worklist, and it's almost boringly standard FHIR.

The hospital opens a case (a `Condition`). The registry exposes a `Task` on a worklist. The hospital posts the `QuestionnaireResponse`; the registry validates it asynchronously and posts back an `OperationOutcome`, the feedback the clinician sees. No new protocol, no vendor-specific mapping layer: a worklist the hospital already knows how to read, expressed in FHIR it already speaks.

![Architecture: on the hospital side, a FHIR Task worklist feeds a pending registration into an SDC form filler pre-populated from EHR/LIS/PACS; the form is submitted to the BCR FHIR server, and validation feedback (valid/invalid) flows back onto the worklist.](/assets/registry-on-fhir-schema.png)

*Everything between the hospital and the registry is plain FHIR: a Task worklist, an SDC form populated from the EHR/LIS/PACS, a submission to the registry's FHIR server, and validation feedback flowing back onto the same worklist.*

And it ran end-to-end. In the demo the worklist is just the doctor's **inbox**: open a task and the form renders straight from the FHIR `Questionnaire` (via the Tiro Web SDK), so the artifact that *defines* the form is the one that *draws* it. We mocked the registry's side — the Belgian Cancer Registry couldn't be in the room — but that's rather the point: the shared FHIR contract let us build both ends without them.

## Why a `Task`, and not just an operation

This is the modeling decision the whole design turns on, and the BCR IG [writes the reasoning down](https://axelv.github.io/bcr/async-validation.html), so it's worth being precise.

The obvious FHIR reflex for "check this for me" is the `$validate` operation. But as the IG puts it, `$validate` *"is structural and blocking, whereas this validation is asynchronous and applies cross-field oncology business rules."* In other words, `$validate` only confirms a resource is well-formed the moment you call it. Whether the registration actually passes the registry's oncology rules is a separate, heavier question: those checks span multiple fields and run as a background job that can take real time.

You could instead wrap that in FHIR's built-in **asynchronous request pattern**: POST with `Prefer: respond-async`, get back `202 Accepted` and a `Content-Location` URL, and poll it until the result is ready (the same machinery Bulk Data runs on, one pattern in R4, split into *Asynchronous Interaction* and *Asynchronous Bulk Data* in R5). But that polling URL is a transient **status monitor**, not a resource: once you fetch the result it's gone. There's nothing left to query, no way for a different actor to discover it, and no trace the attempt ever happened.

A cancer registration needs all of those. So the validation `Task` *is* the durable async job handle: the server returns immediately and validates in the background, but the job stays on the record. As a first-class, queryable resource it hands you a worklist for free (`GET Task?owner=...&status=requested`), `.input`/`.output` to carry the `QuestionnaireResponse` and the result, and Subscriptions to *push* "done" instead of polling (the IG keeps plain polling and topic-based R5-Backport subscriptions both on the table).

The design actually uses **two** Tasks, and the reason is subtle: a long-lived **registration** Task carries the obligation, while a fresh **validation** Task captures each submission attempt. That way neither Task ever has to move backwards. This is the IG's **forward-only** principle and is based on FHIR community consensus: once a Task is terminal it stays terminal, and *"the idiomatic way to 'go back and redo' is a new Task, not a rewound status."* For a legally mandated registry, that history *is* the audit trail.

Two details make this practical. The validation result is an `OperationOutcome` in `Task.output` where each `issue.expression` is a FHIRPath into the submitted `QuestionnaireResponse`, so *"please fix the topography field"* isn't only textual feedback, the UI can point the clinician straight at the offending field. The workflow also never leans on form-data status: `Task.status` tracks the workflow, `QuestionnaireResponse.status` tracks the data, and a correction is just a new submission. No juggling with `amended` status.

(The full resource-level design is published [here](https://axelv.github.io/bcr/async-validation.html), and the test sandbox is on [GitHub](https://github.com/hl7-be/sdc-extract).)

## None of this needed inventing

What's striking is how little of this is new. SDC `$populate`, `Task`, `OperationOutcome`, Subscriptions, every piece already exists in FHIR and every piece is something hospitals are already investing in. This was a two-day proof of concept, and it's an early draft. But it didn't require research. It required wiring things together, and a registry willing to answer.

Which Belgian registry would you want to see talk back in FHIR first?

*Thanks to the teams at the table — Axians, Amaron, UZ Leuven, AZMM — the Belgian Cancer Registry, and Agoria / HL7 Belgium for hosting the FHIR-A-THON.*

#FHIR #StructuredDataCapture #Interoperability #HealthTech #Registries
