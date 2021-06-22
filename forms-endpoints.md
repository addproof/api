# Forms

## Glossary

There are a number of entities that pertain to forms.

* Configurations
* Schemata
* Submissions
* Providers

### Configurations
A configuration is the highest-level construct. Submissions are posted against a configuration. Configurations also specify the automation that occurs when a form is submitted (e.g. who can see the submission; how the parent routing's data is popualted)

Any form configuration has 1 provider.
Any form configuration can have arbitrarily many schemata and submissions.

### Schemata
A submission identifies a version of the form (if only implicitly) and primarily contains the data of the form.

Any version of a form has a schema. A form's schema identifies what questions are on that version of the form, what are their captions, as well as type information about the question (e.g. is this a drop-down). This is not used today for type-checking submission data when it arrives, rather it is used when displaying form submissions to end-users within the app. Readers may see the JSON schema of a form schema [here](schemas/form-structure.yaml).

### Providers

Of all form concepts, that of Provider will be least obvious

A form's provider may be one of FormHero, Drupal or Proof. The provider layer allows for certain simplifications in retrieving schema, correlating schemas with submissions, and mapping schema's from a different form ecosystem into the proof ecosystem.

For forms built on the [FormHero platform](formhero.io), submissions contain information which identifies not only which form the submission came from, but enough information to identify which version of the form was submitted. Because all forms and versions are cloud-hosted, the Proof application is able to call-out and retrieve the schema of a submission upon submission.

For forms built using [Drupal's WebForm](https://www.drupal.org/project/webform), less automation is possible than the FormHero case (as there isn't a single, canonical domain will drupal webforms are hosted), but one can submit a schema as exported from Drupal, and it will be converted into Proof's internal schema format, the schema of which is [here](schemas/form-structure.yaml)


## Endpoints

* [Submit a form](#submit-a-form)
* [Query form submissions](#form-submission-data)
* [Inspect a Schema](#inspect-a-schema)
* [Debug a submission](#debug-a-form)
    - [Example Response](#example-response)
* [Set or Update Form schema](#schemata-set)

## Submit a form 

Form submission endpoints take the form

`/forms(/:provider)(/:form_config_id)/submit`

At least one of `provider` or `form_config_id` must be present.
If `provider` and `form_config_id` are set, they must match.

### Examples: 

* `POST /forms/drupal-webform/submit`
* `POST /forms/drupal-webform/:form_config_id/submit` <- doesn't exist yet, but should
* `POST /forms/form-hero/submit` <- this form is particularly convenient for us for wiring up forms


## Form Submission Data
Work with data submitted via the form.

* `GET forms/121/submissions` will return a paginated list of all the submission for the form configuration `121`

See [form_configs](#) for a list of configured forms.  The [submission debug endpoint](#submission-debug-endpoint) will return the schema of a form submission.

__Optional query parameters:__
 - _page:_ page number to return
 - _per_page:_ number of results per page
 - form data query parameters (see below)

__Form Data Query Parameters__
The API uses path parameters to filter results from submitted form data.  Each query parameter has three elements:
* the type of query: `'filters[email][query]': 'eq`,
* criteria to evaluate `'filters[email][value]': 'me@here.com'`,
* the data type of the criteria `'filters[email][type]': 'string'`,
 

__Supported Query Types:__  
 - starts_with
 - ends_with
 - ilike
 - eq
 - ne
 - gt
 - gte
 - lt 
 - lte 
 - in
 - between 
 
 Currently default is starts_with (for most cases)

__Supported Data Type__

The value of `type` is one of the following:
* `BOOLEAN`
* `DATE`
* `DOUBLE`
* `INTEGER`
* `MONEY`
* `TEXT`
* `VARCHAR`

This is a subset of the postgres data types and is used to interpret the JSON values across all submissions.

Why the Api accepts this type modulation may be unclear.

Readers will be aware that
* the submissions themselves are JSON, so the keys are constrained to be JSON objects
* the form schema could define a stricter set of type valdiations that apply to the key

A few reasons to support this type modulation today include:
* A form's schema is expected to have revisions, some of which may change the type of a question. While we consider it a best-practice to _not_ repurpose an existing question/binding when the type changes, we don't check schemas, upon import, for this kind of consistency across versions to avoid placing this constraint upon form providers.
* The proof application uses the schema to prevent humans from inputting values that deviate from the schema, however when we receive a form submission POSTed to us via API, we do not confirm that the data conforms with the matching schema (if it exists).

We do envision a future state where this type modulation is phased out in favour of stricter type checking across schema versions and upon data POST, but don't have a timeline for this change.

_See the form schema for a details_

```json 
"email_address": {
            "id": "email_address",
            "cuid": "26e442c8-8a31-4d5f-887e-9c71c170c8a0",
            "type": "string",
            "label": "Email address",
            "binding": "email_address"
        },
````

###### Copy as cURL

``` shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" http://https://app.proofgov.com/api/forms/121/submissions?per_page=500&filters%5Bofficeuse.Dateentry%5D%5Btype%5D=date&filters%5Bofficeuse.Dateentry%5D%5Bvalue%5D=2020-09-23&filters%5Bofficeuse.Dateentry%5D%5Bquery%5D=gt
```

## Inspect a Schema

API for schema support is evolving.

Information on the latest schema for a form can be obtained by

* taking the submit URL for the form
* appending `/debug`
* posting an empty JSON body to it.

See the [Debug a form](#debug-a-form) section for more details.


## Debug a form

* `POST /forms/drupal-webform/:form_config_id/submit/debug`

* `POST /api/forms/drupal-webform/:form_config_id/schemata`


__Notes:__
* Has same auth setup as submit endpoint (i.e. no auth - open)
* Does _not_ persist submissions
* Response codes
  - 200 when does a match
  - would 404 if the form_config wasn't found

* Expects the same body format as the submit endpoint
* Returns, in response
  - what schema version we're matching against
  - what are all the question/bindings we know about
  - given the body you submitted, what matched
  - (we're not explicitly telling you what didn't match)

### example response

```
{
    "schema_version": "v3",
    "available_bindings": {
        "applicant.first_name": {
            "id": "applicant_first_name",
            "cuid": "3bb0653e-8f5f-4cec-a7a5-25e911eef9c3",
            "type": "string",
            "label": "Applicant first name",
            "binding": "applicant.first_name"
        },
        "applicant.last_name": {
            "id": "applicant_last_name",
            "cuid": "8d7a0122-ba72-4da9-967a-c8fd67f54deb",
            "type": "string",
            "label": "Applicant last name",
            "binding": "applicant.last_name"
        },
        "applicant.date_of_birth": {
            "id": "applicant_date_of_birth",
            "cuid": "ef1359dd-5d73-4511-8fd9-58f6c72a25e1",
            "type": "date",
            "label": "Applicant date of birth",
            "binding": "applicant.date_of_birth"
        },
        "yukon_resident_resided_in_yukon": {
            "id": "yukon_resident_resided_in_yukon",
            "cuid": "70648789-fd1f-40ed-8bc0-6977d429dd84",
            "type": "dropdown",
            "label": "Did this person reside in Yukon at least 180 days prior to the first day of leave???",
            "binding": "yukon_resident_resided_in_yukon",
            "choices": [
                {
                    "label": "Yes, they resided in Yukon at least 180 days prior.",
                    "value": "yes"
                },
                {
                    "label": "No, they did not reside in Yukon at least 180 days prior.",
                    "value": "no"
                }
            ]
        },
        "email_address": {
            "id": "email_address",
            "cuid": "26e442c8-8a31-4d5f-887e-9c71c170c8a0",
            "type": "string",
            "label": "Email address",
            "binding": "email_address"
        },
        "telephone_number": {
            "id": "telephone_number",
            "cuid": "39b82db5-9be7-42fa-a699-4036ff947ddc",
            "type": "string",
            "label": "Telephone number",
            "binding": "telephone_number"
        },
        "name_of_your_busines": {
            "id": "name_of_your_busines",
            "cuid": "65104004-395e-4e99-8402-ff5555608135",
            "type": "text",
            "label": "What is the name of the business, as registered with Government of Yukon (Corporate Affairs)?",
            "binding": "name_of_your_busines"
        },
        "what_does_your_business_do": {
            "id": "what_does_your_business_do",
            "cuid": "fc81663e-1d57-4c0b-8464-b65711705264",
            "type": "text",
            "label": "What does the business do? For example, services offered or products sold.",
            "binding": "what_does_your_business_do"
        },
        "what_sector_does_your_business_belong_to": {
            "id": "what_sector_does_your_business_belong_to",
            "cuid": "e95a34cd-9c29-453c-a241-3cb78f7d80af",
            "type": "dropdown",
            "label": "What industry sector does the business belong to?",
            "binding": "what_sector_does_your_business_belong_to",
            "choices": [
                {
                    "label": "Agriculture",
                    "value": "agriculture"
                },
                {
                    "label": "Construction",
                    "value": "construction"
                },
                {
                    "label": "Hospitality",
                    "value": "hospitality"
                },
                {
                    "label": "Manufacturing",
                    "value": "manufacturing"
                },
                {
                    "label": "Mining",
                    "value": "mining"
                },
                {
                    "label": "Retail",
                    "value": "retail"
                },
                {
                    "label": "Technology",
                    "value": "technology"
                }
            ]
        },
        "start_date_creation_date_of_your_business": {
            "id": "start_date_creation_date_of_your_business",
            "cuid": "780fb492-9dfa-4f45-b59b-9c428cff7bb6",
            "type": "date",
            "label": "What is the start date (creation date) of the business? Must be before March 1, 2020.",
            "binding": "start_date_creation_date_of_your_business"
        },
        "physical_address.country_code": {
            "id": "physical_address_country_code",
            "cuid": "a8b26502-479c-4931-af58-21cf1ec8e510",
            "type": "string",
            "label": "Physical address: Country code",
            "binding": "physical_address.country_code"
        },
        "physical_address.langcode": {
            "id": "physical_address_langcode",
            "cuid": "bd7b1f15-e836-4ef4-b1ee-7819604bebb1",
            "type": "string",
            "label": "Physical address: Langcode",
            "binding": "physical_address.langcode"
        },
        "physical_address.address_line1": {
            "id": "physical_address_address_line1",
            "cuid": "d66d6ce9-b6d9-4ea2-ac70-549e8db502f2",
            "type": "string",
            "label": "Physical address: Address line1",
            "binding": "physical_address.address_line1"
        },
        "physical_address.address_line2": {
            "id": "physical_address_address_line2",
            "cuid": "c04d8c19-c6dc-4431-ac5b-a80f4399603b",
            "type": "string",
            "label": "Physical address: Address line2",
            "binding": "physical_address.address_line2"
        },
        "physical_address.locality": {
            "id": "physical_address_locality",
            "cuid": "abaccaa2-b5f3-41ee-8b33-7c9ccf47a55e",
            "type": "string",
            "label": "Physical address: Locality",
            "binding": "physical_address.locality"
        },
        "physical_address.administrative_area": {
            "id": "physical_address_administrative_area",
            "cuid": "e4249fbb-fb1e-4f2b-bcfb-835d187c6ccb",
            "type": "string",
            "label": "Physical address: Administrative area",
            "binding": "physical_address.administrative_area"
        },
        "physical_address.postal_code": {
            "id": "physical_address_postal_code",
            "cuid": "0affd15a-30b2-48ef-b536-23aca211e139",
            "type": "string",
            "label": "Physical address: Postal code",
            "binding": "physical_address.postal_code"
        },
        "physical_address.sorting_code": {
            "id": "physical_address_sorting_code",
            "cuid": "c85f1b8d-a193-4714-b028-b7842325736f",
            "type": "string",
            "label": "Physical address: Sorting code",
            "binding": "physical_address.sorting_code"
        },
        "physical_address.dependent_locality": {
            "id": "physical_address_dependent_locality",
            "cuid": "eb0c218c-6350-48a7-bfbb-081b9ebcaccb",
            "type": "string",
            "label": "Physical address: Dependent locality",
            "binding": "physical_address.dependent_locality"
        },
        "mailing_same_as_physical_address": {
            "id": "mailing_same_as_physical_address",
            "cuid": "01f3e207-1237-450d-bc23-c2d04a63b2eb",
            "type": "string",
            "label": "Same as street address?",
            "binding": "mailing_same_as_physical_address"
        },
        "mailing_address.country_code": {
            "id": "mailing_address_country_code",
            "cuid": "67f4f4a4-3593-493e-bbac-84a4075338f4",
            "type": "string",
            "label": "Mailing address: Country code",
            "binding": "mailing_address.country_code"
        },
        "mailing_address.langcode": {
            "id": "mailing_address_langcode",
            "cuid": "493355e0-4bc3-417e-ad31-4d970d16c774",
            "type": "string",
            "label": "Mailing address: Langcode",
            "binding": "mailing_address.langcode"
        },
        "mailing_address.address_line1": {
            "id": "mailing_address_address_line1",
            "cuid": "b4b070df-83a6-48b8-9f27-ce92a80fc651",
            "type": "string",
            "label": "Mailing address: Address line1",
            "binding": "mailing_address.address_line1"
        },
        "mailing_address.address_line2": {
            "id": "mailing_address_address_line2",
            "cuid": "df7bdfe6-12df-4c56-bda6-13c58744ec09",
            "type": "string",
            "label": "Mailing address: Address line2",
            "binding": "mailing_address.address_line2"
        },
        "mailing_address.locality": {
            "id": "mailing_address_locality",
            "cuid": "cc5f0b81-c900-4124-9de6-2edef1d2a083",
            "type": "string",
            "label": "Mailing address: Locality",
            "binding": "mailing_address.locality"
        },
        "mailing_address.administrative_area": {
            "id": "mailing_address_administrative_area",
            "cuid": "28701ba6-a72c-4566-9335-38bac8ea60f8",
            "type": "string",
            "label": "Mailing address: Administrative area",
            "binding": "mailing_address.administrative_area"
        },
        "mailing_address.postal_code": {
            "id": "mailing_address_postal_code",
            "cuid": "9e23cfbd-afae-4227-b160-95af33d3ee71",
            "type": "string",
            "label": "Mailing address: Postal code",
            "binding": "mailing_address.postal_code"
        },
        "date_leave_started": {
            "id": "date_leave_started",
            "cuid": "a8d176b5-5009-4652-af00-1243c21c4e5a",
            "type": "date",
            "label": "When did this leave start?",
            "binding": "date_leave_started"
        },
        "date_leave_ended": {
            "id": "date_leave_ended",
            "cuid": "400b2ff3-4ea8-4c9e-842a-714883653b35",
            "type": "date",
            "label": "When did this leave end?",
            "binding": "date_leave_ended"
        },
        "date_returned_back_to_work": {
            "id": "date_returned_back_to_work",
            "cuid": "dc6cee29-ce41-4296-8bc2-319b7959036e",
            "type": "date",
            "label": "When was the return to work?",
            "binding": "date_returned_back_to_work"
        },
        "how_many_work_days_do_you_want_to_claim_": {
            "id": "how_many_work_days_do_you_want_to_claim_",
            "cuid": "5b7c30f3-0238-4d8e-8c3f-ea1e13ea1970",
            "type": "text",
            "label": "How many work days do you want to claim?",
            "binding": "how_many_work_days_do_you_want_to_claim_"
        },
        "i_am_submitting_this_application_for_the_purpose_of_obtaining_fi": {
            "id": "i_am_submitting_this_application_for_the_purpose_of_obtaining_fi",
            "cuid": "23ab072a-9e49-42d8-982b-1d3813aa8460",
            "type": "string",
            "label": "I am submitting this application for the purpose of obtaining financial assistance from the Government of Yukon for sick leave or self-isolation leave already taken. The statements herein are to the best of my knowledge, true and correct. I affirm that I understand the criteria and intent of the program and am applying to it in good faith.",
            "binding": "i_am_submitting_this_application_for_the_purpose_of_obtaining_fi"
        },
        "i_understand_that_all_or_part_of_this_application_may_be_made_av": {
            "id": "i_understand_that_all_or_part_of_this_application_may_be_made_av",
            "cuid": "e65cc6f3-030f-48fa-a358-bd567a259b63",
            "type": "string",
            "label": "I understand that all or part of this application may be made available to the public in accordance with the Access to Information and Protection of Privacy Act.",
            "binding": "i_understand_that_all_or_part_of_this_application_may_be_made_av"
        },
        "i_understand_the_government_of_yukon_or_its_agents_may_audit_any": {
            "id": "i_understand_the_government_of_yukon_or_its_agents_may_audit_any",
            "cuid": "9376a935-ae9c-4951-a18f-09527354ec02",
            "type": "string",
            "label": "I understand the Government of Yukon or its agents may audit any or all of the records, including financial records of the recipient or its agents as is necessary to satisfy the Government of Yukon that the objectives and activities of the rebate program have been carried out and that the funds have been spent in accordance with the terms of this rebate program.",
            "binding": "i_understand_the_government_of_yukon_or_its_agents_may_audit_any"
        }
    },
    "matches": {
        "applicant.first_name": {
            "id": "applicant_first_name",
            "cuid": "3bb0653e-8f5f-4cec-a7a5-25e911eef9c3",
            "type": "string",
            "label": "Applicant first name",
            "binding": "applicant.first_name"
        },
        "applicant.last_name": {
            "id": "applicant_last_name",
            "cuid": "8d7a0122-ba72-4da9-967a-c8fd67f54deb",
            "type": "string",
            "label": "Applicant last name",
            "binding": "applicant.last_name"
        },
        "applicant.date_of_birth": {
            "id": "applicant_date_of_birth",
            "cuid": "ef1359dd-5d73-4511-8fd9-58f6c72a25e1",
            "type": "date",
            "label": "Applicant date of birth",
            "binding": "applicant.date_of_birth"
        },
        "yukon_resident_resided_in_yukon": {
            "id": "yukon_resident_resided_in_yukon",
            "cuid": "70648789-fd1f-40ed-8bc0-6977d429dd84",
            "type": "dropdown",
            "label": "Did this person reside in Yukon at least 180 days prior to the first day of leave???",
            "binding": "yukon_resident_resided_in_yukon",
            "choices": [
                {
                    "label": "Yes, they resided in Yukon at least 180 days prior.",
                    "value": "yes"
                },
                {
                    "label": "No, they did not reside in Yukon at least 180 days prior.",
                    "value": "no"
                }
            ]
        },
        "email_address": {
            "id": "email_address",
            "cuid": "26e442c8-8a31-4d5f-887e-9c71c170c8a0",
            "type": "string",
            "label": "Email address",
            "binding": "email_address"
        },
        "telephone_number": {
            "id": "telephone_number",
            "cuid": "39b82db5-9be7-42fa-a699-4036ff947ddc",
            "type": "string",
            "label": "Telephone number",
            "binding": "telephone_number"
        },
        "name_of_your_busines": {
            "id": "name_of_your_busines",
            "cuid": "65104004-395e-4e99-8402-ff5555608135",
            "type": "text",
            "label": "What is the name of the business, as registered with Government of Yukon (Corporate Affairs)?",
            "binding": "name_of_your_busines"
        },
        "what_does_your_business_do": {
            "id": "what_does_your_business_do",
            "cuid": "fc81663e-1d57-4c0b-8464-b65711705264",
            "type": "text",
            "label": "What does the business do? For example, services offered or products sold.",
            "binding": "what_does_your_business_do"
        },
        "what_sector_does_your_business_belong_to": {
            "id": "what_sector_does_your_business_belong_to",
            "cuid": "e95a34cd-9c29-453c-a241-3cb78f7d80af",
            "type": "dropdown",
            "label": "What industry sector does the business belong to?",
            "binding": "what_sector_does_your_business_belong_to",
            "choices": [
                {
                    "label": "Agriculture",
                    "value": "agriculture"
                },
                {
                    "label": "Construction",
                    "value": "construction"
                },
                {
                    "label": "Hospitality",
                    "value": "hospitality"
                },
                {
                    "label": "Manufacturing",
                    "value": "manufacturing"
                },
                {
                    "label": "Mining",
                    "value": "mining"
                },
                {
                    "label": "Retail",
                    "value": "retail"
                },
                {
                    "label": "Technology",
                    "value": "technology"
                }
            ]
        },
        "start_date_creation_date_of_your_business": {
            "id": "start_date_creation_date_of_your_business",
            "cuid": "780fb492-9dfa-4f45-b59b-9c428cff7bb6",
            "type": "date",
            "label": "What is the start date (creation date) of the business? Must be before March 1, 2020.",
            "binding": "start_date_creation_date_of_your_business"
        },
        "physical_address.country_code": {
            "id": "physical_address_country_code",
            "cuid": "a8b26502-479c-4931-af58-21cf1ec8e510",
            "type": "string",
            "label": "Physical address: Country code",
            "binding": "physical_address.country_code"
        },
        "physical_address.langcode": {
            "id": "physical_address_langcode",
            "cuid": "bd7b1f15-e836-4ef4-b1ee-7819604bebb1",
            "type": "string",
            "label": "Physical address: Langcode",
            "binding": "physical_address.langcode"
        },
        "physical_address.address_line1": {
            "id": "physical_address_address_line1",
            "cuid": "d66d6ce9-b6d9-4ea2-ac70-549e8db502f2",
            "type": "string",
            "label": "Physical address: Address line1",
            "binding": "physical_address.address_line1"
        },
        "physical_address.address_line2": {
            "id": "physical_address_address_line2",
            "cuid": "c04d8c19-c6dc-4431-ac5b-a80f4399603b",
            "type": "string",
            "label": "Physical address: Address line2",
            "binding": "physical_address.address_line2"
        },
        "physical_address.locality": {
            "id": "physical_address_locality",
            "cuid": "abaccaa2-b5f3-41ee-8b33-7c9ccf47a55e",
            "type": "string",
            "label": "Physical address: Locality",
            "binding": "physical_address.locality"
        },
        "physical_address.administrative_area": {
            "id": "physical_address_administrative_area",
            "cuid": "e4249fbb-fb1e-4f2b-bcfb-835d187c6ccb",
            "type": "string",
            "label": "Physical address: Administrative area",
            "binding": "physical_address.administrative_area"
        },
        "physical_address.postal_code": {
            "id": "physical_address_postal_code",
            "cuid": "0affd15a-30b2-48ef-b536-23aca211e139",
            "type": "string",
            "label": "Physical address: Postal code",
            "binding": "physical_address.postal_code"
        },
        "physical_address.given_name": null,
        "physical_address.additional_name": null,
        "physical_address.family_name": null,
        "physical_address.organization": null,
        "physical_address.sorting_code": {
            "id": "physical_address_sorting_code",
            "cuid": "c85f1b8d-a193-4714-b028-b7842325736f",
            "type": "string",
            "label": "Physical address: Sorting code",
            "binding": "physical_address.sorting_code"
        },
        "physical_address.dependent_locality": {
            "id": "physical_address_dependent_locality",
            "cuid": "eb0c218c-6350-48a7-bfbb-081b9ebcaccb",
            "type": "string",
            "label": "Physical address: Dependent locality",
            "binding": "physical_address.dependent_locality"
        },
        "mailing_same_as_physical_address": {
            "id": "mailing_same_as_physical_address",
            "cuid": "01f3e207-1237-450d-bc23-c2d04a63b2eb",
            "type": "string",
            "label": "Same as street address?",
            "binding": "mailing_same_as_physical_address"
        },
        "mailing_address.country_code": {
            "id": "mailing_address_country_code",
            "cuid": "67f4f4a4-3593-493e-bbac-84a4075338f4",
            "type": "string",
            "label": "Mailing address: Country code",
            "binding": "mailing_address.country_code"
        },
        "mailing_address.langcode": {
            "id": "mailing_address_langcode",
            "cuid": "493355e0-4bc3-417e-ad31-4d970d16c774",
            "type": "string",
            "label": "Mailing address: Langcode",
            "binding": "mailing_address.langcode"
        },
        "mailing_address.family_name": null,
        "mailing_address.address_line1": {
            "id": "mailing_address_address_line1",
            "cuid": "b4b070df-83a6-48b8-9f27-ce92a80fc651",
            "type": "string",
            "label": "Mailing address: Address line1",
            "binding": "mailing_address.address_line1"
        },
        "mailing_address.address_line2": {
            "id": "mailing_address_address_line2",
            "cuid": "df7bdfe6-12df-4c56-bda6-13c58744ec09",
            "type": "string",
            "label": "Mailing address: Address line2",
            "binding": "mailing_address.address_line2"
        },
        "mailing_address.locality": {
            "id": "mailing_address_locality",
            "cuid": "cc5f0b81-c900-4124-9de6-2edef1d2a083",
            "type": "string",
            "label": "Mailing address: Locality",
            "binding": "mailing_address.locality"
        },
        "mailing_address.administrative_area": {
            "id": "mailing_address_administrative_area",
            "cuid": "28701ba6-a72c-4566-9335-38bac8ea60f8",
            "type": "string",
            "label": "Mailing address: Administrative area",
            "binding": "mailing_address.administrative_area"
        },
        "mailing_address.postal_code": {
            "id": "mailing_address_postal_code",
            "cuid": "9e23cfbd-afae-4227-b160-95af33d3ee71",
            "type": "string",
            "label": "Mailing address: Postal code",
            "binding": "mailing_address.postal_code"
        },
        "mailing_address.given_name": null,
        "mailing_address.additional_name": null,
        "mailing_address.organization": null,
        "mailing_address.sorting_code": null,
        "mailing_address.dependent_locality": null,
        "date_leave_started": {
            "id": "date_leave_started",
            "cuid": "a8d176b5-5009-4652-af00-1243c21c4e5a",
            "type": "date",
            "label": "When did this leave start?",
            "binding": "date_leave_started"
        },
        "date_leave_ended": {
            "id": "date_leave_ended",
            "cuid": "400b2ff3-4ea8-4c9e-842a-714883653b35",
            "type": "date",
            "label": "When did this leave end?",
            "binding": "date_leave_ended"
        },
        "date_returned_back_to_work": {
            "id": "date_returned_back_to_work",
            "cuid": "dc6cee29-ce41-4296-8bc2-319b7959036e",
            "type": "date",
            "label": "When was the return to work?",
            "binding": "date_returned_back_to_work"
        },
        "copy_of_most_recent_notice_of_assessment_from_revenue_canada": null,
        "copy_of_tax_benefit_return": null,
        "a_copy_of_your_business_licence_proof_of_your_office_location_or": null,
        "i_am_submitting_this_application_for_the_purpose_of_obtaining_fi": {
            "id": "i_am_submitting_this_application_for_the_purpose_of_obtaining_fi",
            "cuid": "23ab072a-9e49-42d8-982b-1d3813aa8460",
            "type": "string",
            "label": "I am submitting this application for the purpose of obtaining financial assistance from the Government of Yukon for sick leave or self-isolation leave already taken. The statements herein are to the best of my knowledge, true and correct. I affirm that I understand the criteria and intent of the program and am applying to it in good faith.",
            "binding": "i_am_submitting_this_application_for_the_purpose_of_obtaining_fi"
        },
        "i_understand_that_all_or_part_of_this_application_may_be_made_av": {
            "id": "i_understand_that_all_or_part_of_this_application_may_be_made_av",
            "cuid": "e65cc6f3-030f-48fa-a358-bd567a259b63",
            "type": "string",
            "label": "I understand that all or part of this application may be made available to the public in accordance with the Access to Information and Protection of Privacy Act.",
            "binding": "i_understand_that_all_or_part_of_this_application_may_be_made_av"
        },
        "i_understand_the_government_of_yukon_or_its_agents_may_audit_any": {
            "id": "i_understand_the_government_of_yukon_or_its_agents_may_audit_any",
            "cuid": "9376a935-ae9c-4951-a18f-09527354ec02",
            "type": "string",
            "label": "I understand the Government of Yukon or its agents may audit any or all of the records, including financial records of the recipient or its agents as is necessary to satisfy the Government of Yukon that the objectives and activities of the rebate program have been carried out and that the funds have been spent in accordance with the terms of this rebate program.",
            "binding": "i_understand_the_government_of_yukon_or_its_agents_may_audit_any"
        },
        "webform_id": null,
        "_copy_of_most_recent_notice_of_assessment_from_revenue_canada.id": null,
        "_copy_of_most_recent_notice_of_assessment_from_revenue_canada.name": null,
        "_copy_of_most_recent_notice_of_assessment_from_revenue_canada.uri": null,
        "_copy_of_most_recent_notice_of_assessment_from_revenue_canada.mime": null,
        "_copy_of_most_recent_notice_of_assessment_from_revenue_canada.uuid": null,
        "_copy_of_most_recent_notice_of_assessment_from_revenue_canada.data": null,
        "_copy_of_tax_benefit_return.id": null,
        "_copy_of_tax_benefit_return.name": null,
        "_copy_of_tax_benefit_return.uri": null,
        "_copy_of_tax_benefit_return.mime": null,
        "_copy_of_tax_benefit_return.uuid": null,
        "_copy_of_tax_benefit_return.data": null,
        "_a_copy_of_your_business_licence_proof_of_your_office_location_or.id": null,
        "_a_copy_of_your_business_licence_proof_of_your_office_location_or.name": null,
        "_a_copy_of_your_business_licence_proof_of_your_office_location_or.uri": null,
        "_a_copy_of_your_business_licence_proof_of_your_office_location_or.mime": null,
        "_a_copy_of_your_business_licence_proof_of_your_office_location_or.uuid": null,
        "_a_copy_of_your_business_licence_proof_of_your_office_location_or.data": null
    }
}
```

## Schemata Set

* POST /api/forms/drupal-webform/:form_config_id/schemata

* This _is_ authenticated
* Request body
  - must contain schema
  - _may_ contain version (will default to current unix time)

* Only accepts JSON today (could accept YAML)

```
curl --location --request POST 'localhost:3000/api/forms/drupal-webform/1/schemata' \
--header "Authorization: Bearer $PROOF_API_TOKEN" \
--header 'Content-Type: application/json' \
--data-raw '{
    "version": "v1",
    "schema": {
        "start": {
            "#type": "wizard_page",
            "#title": "Start",
            "#next_button_label": "Start a rebate application",
            "start_page_content": {
                "#type": "markup",
                "#markup": "<p>Use this service to apply for a <a href=\"https://yukon.ca/en/your-government/find-out-what-government-doing/paid-sick-leave-rebate-employers#apply-for-a-paid-sick-leave-rebate-for-the-self-employed\">rebate if you are self-employed</a> and need to take time off work due to COVID-19.</p>\n\n<h3>Before you start</h3>\n\n<p>You need:</p>\n\n<ul>\n  <li>A telephone number and email address so we can contact you;</li>\n  <li>a copy of your most recent Notice of Assessment from Canada Revenue Agency (CRA) to provide evidence of self-employment earnings;</li>\n  <li>A copy of the Income and Tax Benefit Return that provides evidence of self-employment income; and</li>\n  <li>a copy of your business licence dated before March 1, 2020, proof of your office location (such as a utility bill, or lease) or municipal business licence.</li>\n</ul>\n\n<p>We will accept a digital photograph or PDF of each document.</p>\n\n<h4>Other ways to apply</h4>\n\n<p>Email <a href=\"mailto:ecdev@gov.yk.ca\">ecdev@gov.yk.ca</a> or phone 867-456-3803 for an application. Phone toll free: 1-800-661-0408, extension 3803.</p>\n\n<hr />\n\n<h4>A note about privacy</h4>\n\n<p>Any personal information you provide on this form is collected for the purpose of administering a funding program pursuant to Section 5 of the&nbsp;<em>Financial Administration Act</em>.</p>\n\n<p>Personal information on this form is collected under the authority of Section 29(c) of the&nbsp;<em>Access to Information and Protection of Privacy (ATIPP) Act</em>&nbsp;for the purpose of carrying out a program and/or providing financial assistance to the applicant.</p>\n\n<p>The collection, use and disclosure of your personal information is managed in accordance with the&nbsp;<em>ATIPP Act</em> and all or part of this information may be made available to the public.</p>\n\n<p>For more information about the collection, use and disclosure of your personal information, please contact:</p>\n\n<p>Department of Economic Development<br />\nATIPP Coordinator/Records Officer<br />\nTelephone: 867-667-5946<br />\n<br />\nOr<br />\n<br />\nDepartment of Finance<br />\nPrivacy Officer/Director, Administration and Systems<br />\nTelephone: 867-667-5933</p>\n"
            }
        },
        "self_employed_applicant_s_information": {
            "#type": "wizard_page",
            "#title": "Self-employed applicant'\''s information",
            "self_employed_applicant_title": {
                "#type": "markup",
                "#markup": "<h2>Self-employed applicant&#39;s information</h2>"
            },
            "self_employed_applicant_description": {
                "#type": "markup",
                "#wrapper_attributes": {
                    "class": [
                        "mrgn-bttm-lg"
                    ]
                },
                "#markup": "<p>Enter the information of the person who will be the primary contact for this application.</p>"
            },
            "applicant": {
                "#type": "composite_simple_person_info",
                "#title": "Applicant",
                "#format_items": "ol",
                "#first_name__required": true,
                "#last_name__required": true,
                "#date_of_birth__required": true
            },
            "yukon_resident_resided_in_yukon": {
                "#type": "radios",
                "#title": "Did this person reside in Yukon at least 180 days prior to the first day of leave???",
                "#options": {
                    "yes": "Yes, they resided in Yukon at least 180 days prior.",
                    "no": "No, they did not reside in Yukon at least 180 days prior."
                }
            },
            "email_address": {
                "#type": "email",
                "#title": "Email address",
                "#description": "<p>A confirmation message will be sent to this email address after your application is received.</p>",
                "#required": true,
                "#format_items": "comma"
            },
            "telephone_number": {
                "#type": "tel",
                "#title": "Telephone number",
                "#description": "<p>For example, 000-000‐0000. We may need to phone you in order to verify details.</p>",
                "#format_items": "comma"
            }
        },
        "your_business_information": {
            "#type": "wizard_page",
            "#title": "Business information",
            "business_information_description": {
                "#type": "markup",
                "#markup": "<p>Enter details about the business.</p>"
            },
            "business_information_title": {
                "#type": "markup",
                "#markup": "<h2>Business information</h2>"
            },
            "name_of_your_busines": {
                "#type": "textfield",
                "#title": "What is the name of the business, as registered with Government of Yukon (Corporate Affairs)?",
                "#required": true,
                "#format_items": "comma"
            },
            "what_does_your_business_do": {
                "#type": "textfield",
                "#title": "What does the business do? For example, services offered or products sold.",
                "#required": true,
                "#format_items": "comma"
            },
            "what_sector_does_your_business_belong_to": {
                "#type": "select_other",
                "#title": "What industry sector does the business belong to?",
                "#options": {
                    "Agriculture": "Agriculture",
                    "Construction": "Construction",
                    "Hospitality": "Hospitality",
                    "Manufacturing": "Manufacturing",
                    "Mining": "Mining",
                    "Retail": "Retail",
                    "Technology": "Technology"
                },
                "#required": true
            },
            "start_date_creation_date_of_your_business": {
                "#type": "datelist",
                "#title": "What is the start date (creation date) of the business? Must be before March 1, 2020.",
                "#required": true,
                "#format_items": "comma",
                "#date_date_max": "2020-03-01",
                "#date_part_order": [
                    "month",
                    "day",
                    "year"
                ],
                "#date_year_range": "1895:2020",
                "#date_year_range_reverse": true,
                "#date_abbreviate": false
            },
            "street_and_mailing_address": {
                "#type": "section",
                "#title": "Street and mailing address",
                "physical_address": {
                    "#type": "address",
                    "#title": "Street address",
                    "#default_value": {
                        "country_code": "CA",
                        "langcode": "",
                        "address_line1": "",
                        "address_line2": "",
                        "locality": "",
                        "administrative_area": "YT",
                        "postal_code": "",
                        "given_name": null,
                        "additional_name": null,
                        "family_name": null,
                        "organization": null,
                        "sorting_code": null,
                        "dependent_locality": null
                    },
                    "#available_countries": [
                        "CA"
                    ],
                    "#field_overrides": {
                        "givenName": "hidden",
                        "additionalName": "hidden",
                        "familyName": "hidden",
                        "organization": "hidden"
                    }
                },
                "mailing_same_as_physical_address": {
                    "#type": "checkbox",
                    "#title": "Same as street address?"
                },
                "mailing_address": {
                    "#type": "address",
                    "#title": "Mailing address",
                    "#states": {
                        "visible": {
                            ":input[name=\"mailing_same_as_physical_address\"]": {
                                "unchecked": true
                            }
                        }
                    },
                    "#default_value": {
                        "country_code": "CA",
                        "langcode": "",
                        "family_name": "",
                        "address_line1": "",
                        "address_line2": "",
                        "locality": "",
                        "administrative_area": "YT",
                        "postal_code": "",
                        "given_name": null,
                        "additional_name": null,
                        "organization": null,
                        "sorting_code": null,
                        "dependent_locality": null
                    },
                    "#format_items": "ol",
                    "#available_countries": [
                        "CA"
                    ],
                    "#field_overrides": {
                        "givenName": "hidden",
                        "additionalName": "hidden",
                        "familyName": "hidden",
                        "organization": "hidden",
                        "sortingCode": "hidden",
                        "dependentLocality": "hidden"
                    }
                }
            }
        },
        "leave_request_and_documents": {
            "#type": "wizard_page",
            "#title": "Leave request and documents",
            "leave_request_and_documents_description": {
                "#type": "markup",
                "#markup": "<p>Enter information about this leave request and attach supporting documents about the business.</p>"
            },
            "information_about_the_leave_request": {
                "#type": "section",
                "#title": "Information about leave request"
            },
            "markup": {
                "#type": "markup",
                "#markup": "This rebate covers dates from March 11, 2020 to September 11, 2020."
            },
            "leave_dates": {
                "#type": "flexbox",
                "date_leave_started": {
                    "#type": "datelist",
                    "#title": "When did this leave start?",
                    "#required": true,
                    "#format_items": "comma",
                    "#date_part_order": [
                        "month",
                        "day",
                        "year"
                    ],
                    "#date_year_range": "2020:2020",
                    "#date_year_range_reverse": true,
                    "#date_abbreviate": false
                },
                "date_leave_ended": {
                    "#type": "datelist",
                    "#title": "When did this leave end?",
                    "#description": "This should be a date in the past.",
                    "#required": true,
                    "#format_items": "comma",
                    "#date_part_order": [
                        "month",
                        "day",
                        "year"
                    ],
                    "#date_year_range": "2020:2020",
                    "#date_year_range_reverse": true,
                    "#date_abbreviate": false
                },
                "date_returned_back_to_work": {
                    "#type": "datelist",
                    "#title": "When was the return to work?",
                    "#description": "If you don&#39;t know the return to work date, ignore this field. We may contact you for information.",
                    "#format_items": "comma",
                    "#date_part_order": [
                        "month",
                        "day",
                        "year"
                    ],
                    "#date_year_range": "2020:2020",
                    "#date_year_range_reverse": true,
                    "#date_abbreviate": false
                }
            },
            "how_many_work_days_do_you_want_to_claim_": {
                "#type": "textfield",
                "#title": "How many work days do you want to claim?",
                "#description": "You cannot claim more than 90 days.",
                "#size": 2,
                "#required": true
            },
            "attach_required_documents": {
                "#type": "section",
                "#title": "Attach documents",
                "#description": "<p>We will accept a digital photograph or PDF of each document.</p>",
                "copy_of_most_recent_notice_of_assessment_from_revenue_canada": {
                    "#type": "managed_file",
                    "#title": "A copy of your most recent Notice of Assessment from Canada Revenue Agency (CRA).",
                    "#description": "We use this to provide evidence of self-employment earnings.",
                    "#required": true,
                    "#format_items": "comma",
                    "#uri_scheme": "s3",
                    "#file_extensions": "gif jpg png bmp tif pdf"
                },
                "copy_of_tax_benefit_return": {
                    "#type": "managed_file",
                    "#title": "A copy of the Income and Tax Benefit Return that provides evidence of self-employment income.",
                    "#description": "<p>For returns from the 2019 tax year, this information can be found on lines 13499 to 14299 and 13500 to 14300. For returns prior to 2019, look at lines 162 to 170 and 135 to 143.</p>",
                    "#required": true,
                    "#format_items": "comma",
                    "#uri_scheme": "s3",
                    "#file_extensions": "gif jpg png bmp tif pdf"
                },
                "a_copy_of_your_business_licence_proof_of_your_office_location_or": {
                    "#type": "managed_file",
                    "#title": "A copy of your business licence, proof of your office location (such as a utility bill, or lease) or municipal business licence.",
                    "#description": "Business licence must be dated before March 1, 2020.",
                    "#required": true,
                    "#format_items": "comma",
                    "#uri_scheme": "s3",
                    "#file_extensions": "gif jpg png bmp tif pdf"
                },
                "declaration": {
                    "#type": "section",
                    "#title": "Before you continue",
                    "declaration_description": {
                        "#type": "markup",
                        "#markup": "You must agree to the following declarations."
                    },
                    "i_am_submitting_this_application_for_the_purpose_of_obtaining_fi": {
                        "#type": "checkbox",
                        "#title": "I am submitting this application for the purpose of obtaining financial assistance from the Government of Yukon for sick leave or self-isolation leave already taken. The statements herein are to the best of my knowledge, true and correct. I affirm that I understand the criteria and intent of the program and am applying to it in good faith.",
                        "#required": true
                    },
                    "i_understand_that_all_or_part_of_this_application_may_be_made_av": {
                        "#type": "checkbox",
                        "#title": "I understand that all or part of this application may be made available to the public in accordance with the Access to Information and Protection of Privacy Act.",
                        "#required": true
                    },
                    "i_understand_the_government_of_yukon_or_its_agents_may_audit_any": {
                        "#type": "checkbox",
                        "#title": "I understand the Government of Yukon or its agents may audit any or all of the records, including financial records of the recipient or its agents as is necessary to satisfy the Government of Yukon that the objectives and activities of the rebate program have been carried out and that the funds have been spent in accordance with the terms of this rebate program.",
                        "#required": true
                    }
                }
            }
        },
        "actions": {
            "#type": "webform_actions",
            "#title": "Submit button(s)",
            "#submit__label": "Finish and submit",
            "#submit__attributes": {
                "class": [
                    "btn-primary"
                ]
            },
            "#wizard_prev__label": "Go back a step",
            "#wizard_next__label": "Continue to next step",
            "#wizard_next__attributes": {
                "class": [
                    "btn-primary"
                ]
            },
            "#preview_prev__label": "Go back a step",
            "#preview_next__label": "Continue to final step",
            "#preview_next__attributes": {
                "class": [
                    "btn-primary"
                ]
            }
        }
    }
}'
```

