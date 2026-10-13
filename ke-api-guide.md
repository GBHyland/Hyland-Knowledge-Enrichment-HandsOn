# Hyland Knowledge Enrichment — Hands-On API Workshop

In this workshop, you'll use **Bruno** to build and send HTTP requests to the Hyland Knowledge Enrichment (KE) APIs. Rather than importing a completed collection, you'll create the environment, requests, and small response scripts yourself. By the end, you'll have a reusable collection that authenticates, uploads content, requests enrichment, and retrieves the results.

> **How KE works:** Knowledge Enrichment is an API service, not a separate document-processing interface. Applications and automated workflows call its APIs to turn unstructured content into useful descriptions, summaries, entities, and metadata.

## Learning objectives

- Configure a Bruno collection and environment for KE.
- Find the correct endpoints and understand their inputs and outputs in the [KE API documentation](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment).
- Authenticate, upload a document, request processing, and retrieve enrichment results.
- Reuse the workflow for a PDF and an image, then optionally try metadata generation.

## Before you begin

1. Install [Bruno - Hyland](https://myhub.hyland.com/search?term=bruno+download&sid=5d9db12a9d3e9a061e321750189a576b&query_origin=typed) -or- [Bruno - Public](https://www.usebruno.com/) on your computer.
2. Ensure you have access to a Hyland Content Innovation Cloud environment, a **Service User**, and an **External Application** authorized for Knowledge Enrichment. If needed, follow this repository's [CIC configuration guide](./cic-config.md).
3. Obtain the external application's **client ID** and **client secret** from your instructor or your own environment.
4. Have a PDF and a JPEG image ready. This repository includes sample content in [Files to Upload](./Files%20to%20Upload/), including `veh-dmg-rep.pdf`.

**Security:** Never commit client secrets, bearer tokens, or presigned URLs to GitHub. Your environment variables for tokens and upload URLs should begin empty. Bruno collection files are stored locally, so review files before committing or sharing them.

## How the requests fit together

| Order | Bruno request | Method | Purpose | Value saved for later |
| --- | --- | --- | --- | --- |
| 1 | Authorize | POST | Exchange application credentials for a token | `auth_token` |
| 2 | HealthCheck (optional) | GET | Check service health | — |
| 3 | Retrieve available actions | GET | Discover supported processing actions | — |
| 4 | PresignedUrl | GET | Get a temporary upload URL and content identifier | `presignedUrl`, `objectKey` |
| 5 | Upload file | PUT | Upload the actual file bytes to storage | — |
| 6 | Content processing | POST | Start enrichment with selected actions | `jobID` |
| 7 | Retrieve result by ID | GET | Check status and read results | — |
| 8 | Content processing - metadata (optional) | POST | Try structured metadata generation | `jobID` |

You'll create these requests in this order. **For the optional exercise, run request 8 and then run request 7 again** to retrieve the new job's results.

---

## Part 1 — Build your Bruno workspace

### 1. Create a collection

1. Open Bruno and choose **Create Collection**.
2. Name it `CIC-KE-Collections`.
3. Choose a folder on your computer where Bruno can save the collection. Open it.
4. Keep collection-level authentication set to **No Auth**. We'll explicitly set Bearer authentication on the KE requests.

A **collection** groups related HTTP requests so you can save, organize, and run them again. Each request below should be created inside this collection.

### 2. Create an environment

1. In your collection, open **Environments** (the environment selector/manager).
2. Create a new environment named `CIC-KE-Environment`.
3. Add the variables shown below, preserving the names and capitalization.
4. Select `CIC-KE-Environment` as the active environment.

| Variable | Initial value | What it does |
| --- | --- | --- |
| `baseURL` | `https://knowledge-enrichment.ai.experience.hyland.com/latest/api/context-enrichment` | Base address for the Context Enrichment API |
| `auth_token` | *(blank)* | Receives a short-lived bearer token |
| `presignedUrl` | *(blank)* | Receives the temporary storage upload URL |
| `objectKey` | *(blank)* | Identifies the uploaded file for processing |
| `jobID` | *(blank)* | Identifies the asynchronous processing job |

Bruno replaces `{{variableName}}` in a request with the value from the active environment. We'll populate most variables automatically using **Post Response** scripts.

> **Checkpoint:** Confirm that your environment is selected before sending any requests. Otherwise, `{{baseURL}}` and the other placeholders will not resolve as expected.

---

## Part 2 — Authenticate and explore KE

### 3. Create `Authorize`

**Why?** KE endpoints require authorization. This request exchanges your application's client credentials for a temporary bearer token.

**API reference:** [Authentication](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment/guides/authentication)

1. Add a new HTTP request named **Authorize**.
2. Set the method to **POST** and the URL to:

   ```text
   https://auth.iam.experience.hyland.com/idp/connect/token
   ```

3. Under **Auth**, select **No Auth**. This request obtains a token; it should not require the token we're about to obtain.
4. Under **Body**, select **Form URL Encoded** and enter:

   | Key | Value |
   | --- | --- |
   | `client_id` | Your external application's client ID |
   | `client_secret` | Your external application's client secret |
   | `grant_type` | `client_credentials` |
   | `scope` | `environment_authorization` |

5. In the **Scripts → Post Response** editor, paste:

   ```javascript
   const response = res.getBody();
   if (response && response.access_token) {
     bru.setEnvVar("auth_token", response.access_token);
     console.log("Authorization token saved to environment.");
   } else {
     console.log("No access_token returned. Check the response.");
   }
   ```

6. Click **Send**. A successful response contains an `access_token`, `token_type`, and `expires_in` value.
7. Check the active environment. `auth_token` should now be populated.

**What happened?** The post-response script read the JSON reply and saved the token. All subsequent KE requests will use `{{auth_token}}`, so you won't need to paste the token repeatedly. Tokens expire; if a later request returns an authorization error, run **Authorize** again.

> **Important:** Do not print the full token in scripts or share screenshots that expose credentials.

### 4. Create `HealthCheck` (optional)

**Why?** This quick request can help distinguish a service-connectivity problem from an authorization or processing problem.

1. Create a **GET** request named **HealthCheck**.
2. URL: `{{baseURL}}/healthy`
3. Choose **No Auth** and **Send**.

The health endpoint is included in the provided Bruno collection. Its behavior can vary by environment; if it isn't available, continue with the documented KE requests.

### 5. Create `Retrieve available actions`

**Why?** Before choosing how to enrich content, ask KE which processing actions it currently supports. This avoids relying on a potentially outdated list.

**API reference:** [Get available process-content actions](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment/guides/testing-your-integration#step-5-check-available-actions)

1. Create a **GET** request named **Retrieve available actions**.
2. URL: `{{baseURL}}/content/process/actions`
3. Under **Auth**, select **Bearer Token** and enter `{{auth_token}}`.
4. Click **Send** and examine the returned list of action names.

**What to notice:** The action names are the exact strings you'll place in the `actions` array when requesting processing. **Use the values returned by your environment**. Older training examples use hyphenated names such as `text-summarization`, while the provided newer Bruno examples use camelCase names such as `imageDescription` and `textMetadataGeneration`. Do not assume the names are interchangeable.

---

## Part 3 — Upload and process a PDF

**API workflow:** [KE documentation — Getting Started](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment)

### 6. Create `PresignedUrl`

**Why?** You don't send a large file in the processing request. First, KE gives you a temporary, signed URL to upload the file into storage, along with an `objectKey` to identify it later.

**API reference:** [Get a presigned URL for uploading a file](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment/Reference/Context%20API/Endpoints/get-a-presigned-url-for-uploading-a-file)

1. Create a **GET** request named **PresignedUrl**.
2. URL: `{{baseURL}}/files/upload/presigned-url`
3. Under **Params / Query**, add `contentType` with the value `application/pdf`.
4. Under **Auth**, select **Bearer Token**: `{{auth_token}}`.
5. Under **Scripts → Post Response**, paste:

   ```javascript
   const response = res.getBody();
   if (response && response.presignedUrl && response.objectKey) {
     bru.setEnvVar("presignedUrl", response.presignedUrl);
     bru.setEnvVar("objectKey", response.objectKey);
     console.log("Upload URL and object key saved.");
   } else {
     console.log("Upload URL or object key missing.");
   }
   ```

6. Click **Send**. Look for `presignedUrl` and `objectKey` in the JSON response and confirm both environment variables were populated.

**What happened?** `presignedUrl` is a short-lived, signed storage address; `objectKey` is the identifier you'll pass to KE for processing. The upload URL is **not** the KE API base URL.

### 7. Create `Upload file`

**Why?** This request transfers the actual PDF bytes to the temporary upload location. Getting a presigned URL alone does not upload anything.

1. Create a **PUT** request named **Upload file**.
2. URL: `{{presignedUrl}}`.
3. Set **Auth** to **No Auth**. The presigned URL already contains its own authorization information; **do not send the KE bearer token to the storage URL**.
4. Under **Body**, select **File** (binary file upload) and choose `veh-dmg-rep.pdf` from [Files to Upload](./Files%20to%20Upload/).
5. If Bruno allows you to set the file content type, use `application/pdf`. Ensure any `Content-Type` header matches the MIME type requested when generating the URL.
6. Click **Send**. Look for a successful 2xx response (commonly **200 OK**). A successful upload may have an empty response body.

**Important:** Don't change or manually re-encode the signed query string. If the URL expires or the upload fails with a signature-related error, run **PresignedUrl** again and retry promptly. The uploaded file and `objectKey` must come from the same presigned-URL request.

### 8. Create `Content processing`

**Why?** Uploading a file only stores it. This request tells KE **which uploaded file** to process and **which enrichment actions** to run.

**API reference:** [Process content asynchronously](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment/Reference/Context%20API/Endpoints/process-content-asynchronously)

1. Create a **POST** request named **Content processing**.
2. URL: `{{baseURL}}/content/process`.
3. Set **Auth → Bearer Token** to `{{auth_token}}`.
4. Under **Body → JSON**, enter the following. The action names below are illustrative: confirm the supported text actions from **Retrieve available actions** and replace them if your environment uses different names.

   ```json
   {
     "objectKeys": ["{{objectKey}}"],
     "actions": ["text-summarization", "named-entity-recognition-text"]
   }
   ```

5. Under **Scripts → Post Response**, add:

   ```javascript
   const response = res.getBody();
   if (response && response.processingId) {
     bru.setEnvVar("jobID", response.processingId);
     console.log("Processing job ID saved.");
   } else {
     console.log("No processingId returned. Check the response.");
   }
   ```

6. Click **Send**. A successful submission returns a `processingId` (the exact response may contain other fields).
7. Confirm that `jobID` is populated in your environment.

**What happened?** KE accepted an asynchronous job. The `processingId` is not the enrichment result; it is the identifier used to check on the job later.

> **Action-name checkpoint:** If the server rejects an action, inspect the results of **Retrieve available actions**. The provided Bruno sample currently uses `imageDescription` for an image, while the original guide used hyphenated text actions for a PDF. Use the names that your live KE environment actually returns.

### 9. Create `Retrieve result by ID`

**Why?** Processing happens asynchronously. This request checks the job and, when it finishes, retrieves the enriched output.

**API reference:** [Get content-processing results](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment/Reference/Context%20API/Endpoints/get-content-processing-results)

1. Create a **GET** request named **Retrieve result by ID**.
2. URL: `{{baseURL}}/content/process/{{jobID}}/results`.
3. Set **Auth → Bearer Token** to `{{auth_token}}`.
4. Click **Send**.
5. If the response indicates the job is still processing (for example, `status: PROCESSING`), wait a few moments and send it again.
6. Once complete, inspect the JSON for the outputs corresponding to the actions you requested. Look for the summary and extracted entities if those actions are supported and succeeded.

**Checkpoint:** You have now built a complete flow: **authenticate → get upload URL → upload PDF → request processing → retrieve results**.

---

## Part 4 — Reuse the requests for an image

You'll now run the same sequence without building another collection. This illustrates how a single integration can process different kinds of content.

### 10. Upload a JPEG image

1. Run **Authorize** again if your token has expired.
2. Open **PresignedUrl** and change the `contentType` query parameter from `application/pdf` to `image/jpeg` (the provided Bruno sample uses `image/jpg`; use a MIME type supported by your environment and keep it consistent for the upload).
3. Click **Send** to obtain a **new** `presignedUrl` and `objectKey`.
4. Open **Upload file** and replace the PDF with a JPEG from [Files to Upload](./Files%20to%20Upload/). Use the same image MIME type as the presigned-URL request.
5. Click **Send** and confirm the upload succeeded.

### 11. Request image enrichment

1. Open **Content processing**.
2. Replace the JSON body with the following example, **provided `imageDescription` is listed among the available actions**:

   ```json
   {
     "objectKeys": ["{{objectKey}}"],
     "actions": ["imageDescription"]
   }
   ```

3. Click **Send**. The existing script saves the new `processingId` into `jobID`.
4. Open **Retrieve result by ID** and click **Send**. Repeat after a short wait if processing is still underway.
5. Review the generated description. What did KE identify about the image? What useful information could another application take from this response?

**Explore:** If the live actions list includes an image entity-recognition action, try adding it to the `actions` array and compare the output. The exact supported name and response structure are determined by your environment.

---

## Part 5 — Optional advanced exercise: Generate structured metadata

**Goal:** Explore how KE can produce metadata guided by example fields. This is useful when an application needs structured values for search, filtering, classification, or downstream automation rather than a free-form description.

**API reference:** [Process content asynchronously](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment/Reference/Context%20API/Endpoints/process-content-asynchronously)

### 12. Create `Content processing - metadata`

1. First, upload an appropriate image or document using **PresignedUrl** and **Upload file**. This gives you a fresh `objectKey`.
2. Run **Retrieve available actions** and confirm that `textMetadataGeneration` is supported before proceeding. The provided Bruno collection uses this action name.
3. Create a **POST** request named **Content processing - metadata**.
4. URL: `{{baseURL}}/content/process`.
5. Set **Auth → Bearer Token** to `{{auth_token}}`.
6. Under **Body → JSON**, enter the sample structure from the provided Bruno request:

   ```json
   {
     "objectKeys": ["{{objectKey}}"],
     "actions": ["textMetadataGeneration"],
     "kSimilarMetadata": [
       {
         "claim": {
           "manufacturer": "Pontiac",
           "model": "Firebird",
           "color": "red",
           "year": "1992",
           "car_part": "bumper",
           "damage_type": "minimal",
           "damage_severity": "low",
           "confidence_score": "10"
         }
       }
     ]
   }
   ```

7. Add the same **Post Response** script used in **Content processing** to save `processingId` as `jobID`.
8. Click **Send**, then run **Retrieve result by ID** until the job completes.

**What to notice:** `kSimilarMetadata` supplies an example metadata shape and values, which may help guide metadata generation. These values are **examples, not verified facts about the uploaded content**. Inspect the returned fields and assess their accuracy before using them in business processes. This example is drawn from the attached Bruno collection; validate the payload against the documentation and your environment if the endpoint rejects it.

**Discuss:** How might generated fields such as vehicle make, model, damaged part, and damage severity help an insurance claims application? Which fields should a person verify before using them to make a decision?

---

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Bruno says a variable is undefined | Confirm the correct environment is active and the variable name matches exactly. |
| 401 / unauthorized | Rerun **Authorize**; verify the service user, external application, credentials, and permissions. |
| 400 / unsupported action | Run **Retrieve available actions** and use an exact supported action name. |
| Upload fails with 403 / signature error | Generate a fresh presigned URL; do not alter its query string; verify the upload MIME type. |
| `getaddrinfo ENOTFOUND undefined` | Check that `{{presignedUrl}}` resolves to a full `https://...` URL. Don't send the PUT request until the presigned-URL request has succeeded and populated the variable. |
| PUT upload returns an empty body | This can be normal; check the HTTP status instead. |
| Result still says `PROCESSING` | Wait and retry **Retrieve result by ID**. |
| Metadata results are unexpected | Check that the uploaded file fits the requested action and that example metadata isn't being mistaken for verified output. |

### Finished!

You've created a reusable Bruno workflow that demonstrates how applications interact with Knowledge Enrichment APIs. The same pattern can be integrated into an application or automation: obtain a token, upload content, start enrichment, and consume the structured results.

**Further reading:** [Knowledge Enrichment documentation](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment) · [This workshop's GitHub repository](https://github.com/GBHyland/Hyland-Knowledge-Enrichment-HandsOn)
