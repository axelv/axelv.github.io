---
title: "Template-based extraction "
author: "Axel Vanraes"
categories:
  - "FHIR"
  - "SDC"
  - "forms"
---
## Use cases

### 1. Sugery report Robotic-assisted laparoscopic prostatectomy

A first use-case is a surgery report for a robotic-assisted laparoscopic prostatectomy (RALP). The Questionnaire below is actually a simplified fragement of the real template. It is used to report the surgery details and specifically the additional procedures that were performed during the surgery.

Questionnaire:

```yaml
resourceType: Questionnaire
title: "Surgery report RALP"
url: http://templates.tiro.health/templates/surgery-report-ralp
status: active
item:
  - linkId: additional-procedures
    text: "Additional procedures"
    type: choice
    repeats: true
    answerOptions:
      - valueCoding:
          display: "Pelvic lymph node dissection"
          code: "PLND"
      - valueCoding:
          display: "Seminal vesicle dissection"
          code: "SVD"
      - valueCoding:
          display: "Bladder neck reconstruction"
          code: "BNR"
      - valueCoding:
          display: "Urethral reconstruction"
          code: "UR"
    item:
      - linkId: laterality
        text: "Laterality"
        type: choice
        enabledWhen:
          - question: "#additional-procedures"
            operator: =
            answerCoding:
              code: "PLND"
        answerOptions:
          - valueCoding:
              display: "Left"
              code: "L"
          - valueCoding:
              display: "Right"
              code: "R"
          - valueCoding:
              display: "Bilateral"
              code: "B"
extension:
  - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractToTemplate
    valueReference:
      reference: "#extraction-bundle"
contained:
  - resourceType: Bundle
    type: transaction
    entry:
      - fullUrl: "urn:uuid:ralp"
        request:
          method: POST
          url: "Procedure"
        resource:
          resourceType: Procedure
          status: completed
          subject:
            extension:
              - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                valueString: "%resource.subject"
            display: Current subject
          recorded: "1970-01-01"
          _recorded:
            extension:
              - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                valueString: "%resource.authored"
          code:
            text: "Robotic-assisted laparoscopic prostatectomy"
      - extension:
          - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
            valueExpression:
              language: text/fhirpath
              expression: "%resource.item.where(linkId='additional-procedures')"
        request:
          method: POST
          url: "Procedure"
        resource:
          resourceType: Procedure
          status: completed
          subject:
            extension:
              - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                valueString: "%resource.subject"
            display: Current subject
          recorded: "1970-01-01"
          _recorded:
            extension:
              - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                valueString: "%resource.authored"
          code:
            text: "Additional procedure"
            _text:
              extension:
                - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                  valueString: answer.valueCoding.display
            coding:
              - extension:
                  - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                    valueString: "answer.valueCoding"
          partOf:
            - reference: #ralp
          bodySite:
            - coding:
                - extension:
                    - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                      valueString: "answer.valueCoding"
```


### 2. Repeating cancer stagings

A second more complex use-case to explore is the staging of lung cancer. The reduced template below is used for MDT-meetings where the staging of lung cancer is discussed. The template is used to report the mulitple presentations of the same patient over time.

Questionnaire:

```yaml
resourceType: Questionnaire
title: Lung Cancer presentation
url: http://templates.tiro.health/templates/mdt-lung-cancer
status: active
item:
  - linkId: presentation
    text: "Lung Cancer presentation"
    repeat: true
    type: group
    item:
      - linkId: date-of-diagnosis
        text: "Date of diagnosis"
        type: date
        required: true
      - linkId: diagnosis
        text: "Stage"
        type: choice
        required: true
        answerValueSet: "http://terminology.tiro.health/r5/ValueSet/lung-cancer-diagnosis"
      - linkId: tnm-staging
        text: "TNM staging"
        type: group
        item:
          - linkId: t
            text: "T-category"
            type: choice
            answerValueSet: "http://terminology.tiro.health/r5/ValueSet/lung-cancer-tnm-t-category"
          - linkId: n
            text: "N-category"
            type: choice
            answerValueSet: "http://terminology.tiro.health/r5/ValueSet/lung-cancer-tnm-n-category"
          - linkId: m
            text: "M"
            type: choice
            answerValueSet: "http://terminology.tiro.health/r5/ValueSet/lung-cancer-tnm-m-category"
extension:
  - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractToTemplate
    valueReference:
      reference: "#extraction-bundle"
contained:
  - resourceType: Bundle
    type: batch
    entry:
      - extension:
          - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
            valueExpression:
              language: text/fhirpath
              expression: "resource.item.where(linkId='presentation')"
        resource:
          resourceType: Condition
          status: active
          subject:
            extension:
              - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                valueString: "%resource.subject"
            display: Current subject
          code:
            text: "Diagnosis"
            _text:
              extension:
                - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
                  valueString: "Diagnosis"
      - extension:
          - url: http://hl7.org/fhir/uv/sdc/StructureDefinition/sdc-questionnaire-extractTemplateValue
            valueExpression:
```

QuestionnaireResponse:

```yaml
resourceType: QuestionnaireResponse
status: completed
subject:
  reference: Patient/123
encounter:
  reference: Encounter/456
authored: "2024-11-10"
item:
  - linkId: presentation
    item:
      - linkId: date-of-diagnosis
        answer:
          - valueDate: "2023-01-11"
      - linkId: diagnosis
        answer:
          - valueCoding:
              display: "Non-small cell lung cancer"
              system: "http://snomed.info/sct"
              code: "254637007"
      - linkId: tnm-staging
        item:
          - linkId: t
            answer:
              - valueCoding:
                  code: "cT1"
                  display: "Tumor stage 1"
          - linkId: n
            answer:
              - valueCoding:
                  code: "cN0"
                  display: "No regional lymph node metastasis"
          - linkId: m
            answer:
              - valueCoding:
                  code: "cM0"
  - linkId: presentation
    item:
      - linkId: date-of-diagnosis
        answer:
          - valueDate: "2024-11-10"
      - linkId: diagnosis
        answer:
          - valueCoding:
              display: "Non-small cell lung cancer"
              system: "http://snomed.info/sct"
              code: "254637007"
      - linkId: tnm-staging
        item:
          - linkId: t
            answer:
              - valueCoding:
                  code: "cT2"
                  display: "Tumor stage 2"
          - linkId: n
            answer:
              - valueCoding:
                  code: "cN1"
                  display: "Regional lymph node metastasis"
          - linkId: m
            answer:
              - valueCoding:
                  code: "cM0"
                  display: "No distant metastasis"





## Thoughts
- It's a little too expect fields to be completed without guarantees. The extraction process however will succeed because of the default dummy values. Should there be a way to explicitly define expectation about optoinality and cardinality?

-
