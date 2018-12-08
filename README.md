# Overview

We see a citizen-facing form as presenting a user with series of questions (prompts) and collects answers as well as some metadata (i.e. IP of submitter, browser, OS, etc.). This data - the prompts presented to a user, the answers a user gave to a prompt, and metadata - must be communicated to a back-office system. The protocol and structure of this interaction is the primary subject of this document.

This API is designed so that form structure - # of questions, prompting text, variation of possible answers - may be modified over an integration lifetime. The intention is to give flexibility to the designers of both front- and back-office systems by reducing unneccessary coordination and coupling. We understand that coordination around some certain questions will be necessary in some cases (i.e. it is envisioned that more sophisticated forms may need to validate a user's current input against past applications) and we see this as easy within this setup; we'll spend some time discussing the details of such an extension in a later section.

Our API is a RESTful JSON API that may be interacted with via HTTPS. Authentication will be done via bearer token to be provided via a header (X-Auth) of any request; the token will be communicated out of band; we defer use of AD service accounts to a later phase. We envision a future where clients of the front-office system may submit directly to the back-office system using a session-specific secret, but we defer this to a later phase.

The end-points for the API are:

1. /forms/LPN-new-application/sessions                      (POST here when a new session starts)
2. /forms/LPN-new-application/sessions/:session-id/answers  (POST here as new data is submitted)
3. /forms/LPN-new-application/sessions/:session-id/files    (POST here when files are uploaded; direct posts from the client box to be supported in a future release)
4. /forms/LPN-new-application/sessions/:session-id/metadata (POST here if new client browser metadata is collected over session life)
5. /forms/LPN-new-application/sessions/:session-id/status   (PUT here to signal user's submission)

## Example work-flow

1. A client (i.e. eService/Kellett/Drupal/LPN-form-submission-system) POSTs to the /sessions end-point when a user lands on the page; this POST contains some client browser metadata. A session-id is generated by the back-office system; this user's session cookied with this id.
2. When the user clicks "Next" to move through a page of questions, the client POSTs to the /sessions/:session-id/answers end point with the answers on the last page. The back-office returns a 204 (OK, no content in response). This step repeats as the user moves through the form.
3. The client PUTs /sessions/:session-id/status with '{ "value": "complete" }' when the client clicks submit.

We also permit the client to assemble all data and submit it with a single POST to /sessions upon form completion (but encourage the adoption of the more incremental session application)

## Planned extensions

* support authentication via AD service accounts
* generate a per-session bearer token to accept posts directly from a client box
* create more flexibility around communicating field types and prompts (e.g. create a /forms/:form_id/:fields endpoint to manage fields for the form, rather than repeating this at each user submission)

# Comprehensive end-point list + JSON Schema

This section gives more details on the various endpoints of the API. Schema for requests and responses, when they exist, are found in the /schemas folder. Again, the list of API end-points is as follows:

1. /forms/LPN-new-application/sessions                      (POST here when a new session starts)
2. /forms/LPN-new-application/sessions/:session-id/answers  (POST here as new data is submitted)
3. /forms/LPN-new-application/sessions/:session-id/files    (POST here when files are uploaded)
4. /forms/LPN-new-application/sessions/:session-id/metadata (POST here if new client browser metadata is collected over session life)
5. /forms/LPN-new-application/sessions/:session-id/status   (PUT here to signal user's submission)

All the endpoints in this section will be mounted at a prefix like /forms/LPN-new-application. JSON received and served in production will be validated according to the following schema documents:

## /sessions

* GET will return a list of recent session ids.
* POST will initiate a session. Schema 

## /sessions/:session-id/answers

* GET will return the list of field ids for which answers have been submitted, as well as time of answer (to disambiguate multiple submissions), _but will not include answer content_; we aim to support integration debugging but with a model with good defaults for privacy and security. This proposal means regardless of the question, should an citizen-facing eServices form (available on the public internet) be compromised, this will not compromise user data.

example response:

```json
{
  "answers": [
    {
      "field_id": "Y6Q2OdRmpuLQ",
      "updated_at": "2018-06-04 11:15:32-07"
    },
    {
      "field_id": "lwXFHFNjwcnz",
      "updated_at": "2018-06-01 15:16:45-07"
    }
  ]
}
```

POSTs to this end-point will record new answers for that session. An answer specifies at least

* a field id
* a field type
* a prompting text
* a value

Some other properties will also be added for certain field_type (e.g. if it's a multiple-choice or multi-select question, the other choices should be specified).

An explicit goal of this API is to minimize coordination between front and back office systems for forms requiring no user-specific validation. By allowing the front-office facing form to push prompting text, we free the front-office system to iterate on form design and even post launch and things should "just work".

It is of course redundant for this prompting text to be submitted with each submission by a user (once a form goes live, it is unlikely to receive regular revisions). This redundancy can be factored out in a later stage (e.g. by exposing a /forms/:form_slug/fields endpoint or by exposing a /field-details endpoint on the citizen-facing system)


example request

```json
[
  {
    "field_id": "Y6Q2OdRmpuLQ",
    "field_type": "string",
    "prompt": "Please enter your full legal name",
    "value": "Wesley George"
  },
  {
    "field_id": "lwXFHFNjwcnz",
    "field_type": "choices",
    "prompt": "Have you worked as a nurse in the last 10 years?",
    "value": "No I have not",
    "choices": [
      "Yes I have",
      "No I have not",
    ],
  },
  {
    "field_id":   "n7FYOGpx36D4",
    "field_type": "table[7]",
    "prompt": "Please enter the details of your recent employers",
    "headings": [ "name of business", "business phone #", "supervisor name", "supervisor phone #", "supervisor email", "start date", "end date" ],
    "value": [
      [
        "Victoria General Hospital",
        "250-727-4212",
        "Mr. Bertilda Loggins",
        "250-883-3423",
        "bertralilda.loggilady@gmail.com",
        "2017-01-15",
        "2017-07-22"
      ],
      [
        "Whitehorse General Hospital",
        "(867) 393-8700",
        "Esmerelda Huneyducks",
        "867-453-3663",
        "mrs.huneyducks@hotmail.com",
        "2017-09-22",
        "2018-06-01"
      ],
    ]
  },
]
```

Notes:
  * This end-point has the most refined schema. Consult the /schemas/requests/answers.POST.json for details.
  * Clients should select field-ids that identify a question within a form and use the same id consistent when answers to that question are provided from different sessions.
  * A client is not restricted from changing an answer to a question - if the same field id appears across multiple POST requests, the last one processed will be treated as the answer submitted by the user.
  * Fields have a type which should not change. If multiple types are submitted for the same field id - even across different sessions - the server may return 422 (conflict).
  * We envision files handling may entirely be handled by the `/files` endpoint. If there is a need to provide more form context than a descriptive filename, clients may choose to use the the `file_ref` field type, which allows them to specify the mapping between a form prompt and a _previously uploaded file_.

## /sessions/:session-id/metadata

Endpoint available for a client to update user browser metadata beyond initial session establishment. It is expected, though not an explicit pre-condition for any part of the system, that user's session will occur in a single browser.

## /sessions/:session-id/files

Endpoint for uploading files to an application. We support typical form uploads (i.e. Content-Type: multipart/form-data), as well as supporting JSON posts with the schema

```
{
  type: 'object',
  properties: {
    filename: { type: 'string' },
    file_mime: { type: 'string'},
    file_data: { type: 'string', description: 'base64 encoding of filedata'},
  }
  required: [ 'filename', 'file_data', 'file_mime' ]
}
```

The response to this end point will satisfy the following schema:

```
{
  type: 'object',
  properties: {
    file: {
      type: 'object',
      required: [ "id" ],
    }
  }
}
```

The returned id may be used as the value of an answer of type file_ref to give more context on the uploaded file, e.g.
```
[
  {
    field_id: "government-id",
    field_type: "file_ref",
    prompt: "Please upload a copy of your government id",
    value: 14,
  },
]
```

## /sessions/:session-id/status

Endpoint used to communicate application form finalization (e.g. when the user pushes 'submit')

# Feedback

Feedback is solicited, particularly with regard to field types.

This API draws considerable inspiration from the API for [Typeform](typeform.com), but aiming to make it as simple as possible to map an arbitrary form into a format comprehensible by the back-end system. The most significant departure from thence is introduction of the 'table' data type, to acknowledge that it is common, on government forms, to solicit data from users structured in this way (where an a-priori unknown number of answers need to be given, but all have the same structure)

# Extension case: front/back-office coordination for user-specific input validation

## Examples where coordination is be necessary

1. Bob is applying to renew his nursing license. It is his 4th year renewing, so his fifth year of practice as a Nurse in the Yukon, so more detail is needed concerning his continuing education activities over the past 5 years.
2. Douglas is applying to renew his nursing license. His account is owing due to some past fines or problems with book-keeping, so the amount of money paid during registration is hire.

## Precondition: User-Authentication 
Coordination, such as the example above, presumes that the front-office systems knows the identity of the user at the other end of a session, which is user authentication. A YG-wide implementation of such browser-based user authentication is a roadmap item for ICT. From the perspective of this author, this item is not slated for deliver in the immediate future (time of writing is June 4th, 2018).

## Example Accomodation

* A /validations end point is introduced, which exists in the context of a session, e.g. a client could interact with a URL such as /forms/LPN-application/sessions/:session_id/validations.
* requests to that endpoint use the same schema as the `/answers` endpoint.
* Front and back office systems would need to agree upon field ids for questions needing validation (today, there is no need for the front-office to reliquich control over a field id).

Likely there would be some categories of common remediation, which should be reflected in schema for `/validations` similar to `/answers`.



