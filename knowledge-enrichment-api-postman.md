# Hyland Knowledge Enrichment API Service

This guide is meant to walk you through step-by-step on leveraging Hyland Knowledge Enrichment APIs from a developer perspective using an HTTP tool (Postman).

**Reference.** All API references herein can be found at [Hyland's Knowledge Enrichment Github Page](https://hyland.github.io/ContentIntelligence-Docs/KnowledgeEnrichment/Overview).

---

### Set Up Postman Environment
We'll perform a few steps to configure the Postman application with environment variables and collections.
1. Visit [this page](./Postman%20Files/) to download the Postman imports for the _environment_ and _collections_.
2. Launch Postman.
   - If you do **not** have a Postman account, you must create one for this class (sorry!). It's free! You should see an account button in the top-right corner of the app.
   - If you have an account, please ensure you are signed in (top-right corner of the postman app).
3. On the left panel, **Collections** should be selected (select it if not). Click on the **Import** button above the left panel and upload the file downloaded in **step 1** titled ```CIC-KE-Collections```.
4. On the left panel, select **Environment**. Click on the **Import** button above the left panel and upload the file downloaded in **step 1** titled ```CIC-KE-Environment```.

---

### Send HTTP Requests to the Knowledge Enrichment APIs:
Using Postman, we'll send requests directly to the Knowedge Enrichment APIs.
You'll recall that sending a document for data enrichment requires multiple API requests sent in a specific order. Here's that breakdown once again:
| Request       | Data Params       | Returns       |
| ---           | ---               | ---           |
| Authorization | client_id, client_secret | Bearer token |
| Get Presigned URL | Bearer token  | Presigned URL & ObjectKey |
| Upload doc    | presigned URL     | nothing       |
| Content Processing    | Bearer token, ObjectKey, actions    | JobID |
| Request Results   | Bearer token, JobID | Results (JSON)    |

<br>

**Get Authorization Credentials from Auth API:**  
First, we'll request authorization to connect to the Knowledge Enrichment API.
1. In Postman, ensure that **Collections** is selected from the left panel. You should see a catalogue of collections titled _CIC-KE-Collections_. 
2. Toggle the _CIC-KE-Collections_ down to reveal the list of HTTP Requests. 
3. Select the request titled ```POST Authorize```. 
4. This will open the HTTP Request in the main area of Postman.
5. Under the URL field you'll see horizontally placed tabs titled: ```Params```, ```Authorization```, ```Headers```, ```Body```, etc. _Select the **BODY** tab._
6. The Body tab holds key-value pairs that must be sent to the API. Notice that the values for ```client_id``` and ```client_secret``` are missing. **Paste in your client_id and client_secret accordingly**.
5. Press the blue **Send** button.
You should receive a JSON reply similar to the following:
```
{
    "access_token": "... a long token string ...",
    "expires_in": 900,
    "token_type": "Bearer",
    "scope": "environment_authorization"
}
```
| Note  |
| ---   |
| The access_token is provided and must be used to make further calls to the API. <br> In this Postman environment, a script is in place to catch the Bearer Token value and place it into an environment varibale that is baked into the other HTTP Requests; so there's no need to copy and paste this value!     |  



**Get a list of actions from the API:**
Next, we'll send a simple **GET** request that will ask the API to return a list of actions that we may request to perform on uploaded documents. This ensures that we have the most up-to-date actions from the API itself.
1. From the list of collections, select the request titled ```GET Receive available actions```. 
   - This will open the new HTTP Request into a new tab in the main portion of the app.
2. Press the blue **Send** button.
You should receive a JSON array reply of the available actions you may perform that should look like the following:
```
[
  "image-description",
  "image-metadata-generation",
  "text-metadata-generation",
  "text-classification",
  "text-summarization",
  "image-classification",
  "image-embeddings",
  "text-embeddings",
  "named-entity-recognition-image",
  "named-entity-recognition-text"
]
```
| Note  |
| ---   |
| You now have a list of the most recent available actions. Later, when requesting the API to process an uploaded document you may choose to include one or more of these actions at your leisure.     | 

---

### Send a document for Summarization:

**Get a Presigned URL**
Sending a document for enrichment requires multiple API calls as outlined above. We'll run through a quick series of API calls to get a document summarization.
The API expects the following:
- Bearer Token
- Content Type Param set to expected content type of file to be uploaded

1. You should already have a Request tab opened for the ```POST Authorize``` Request. _Select that tab_.
2. Press the blue **Send** button to get a new Bearer Token (This will refresh our Bearer Token since it expires).
3. From the list of collections, select the request titled ```GET PresignedURL```. 
4. Press the blue **Send** button.
Once successfully executed, you'll get a JSON return with two key data values. One will be the _presignedUrl_ and the other will be an _objectKey_. It will look like this:
```
{
  {
    "presignedUrl": "https://a-long-url/",
    "objectKey": "contents/4c9b66de-fc0b-4c43-8fb1-40efb689bf72/ff49c0dd-5fd2-4b57-8f19-a6cf0372e2d9/ff49c0dd-5fd2-4b57-8f19-a6cf0372e2d9"
}
}
```
**Explanation:**
- **PresignedURL:** The URL we will use to upload our document to the S2 storage bucket.
- **ObjectKey:** The identifier for this document in the S3 bucket. We'll use this ID next to tell the context API to process this document so it knows what document to process.


**Upload a Document to the S3 Bucket**
Uploading a document is a straight-forward process once you've received a _presigned URL_. 
The API expects the following:
- Bearer Token
- The Presigned URL
- The Content Type expect to be uploaded. 

1. From the list of collections, select the request titled ```PUT Upload File```. 
2. From the tabs located below the HTTP location field, select **Body**.
3. Click on the empty **Select file** field to open a file prompt.
4. Navigate to this path from the root level of the files provided: ```/Files to Upload/``` and select the file titled: ```veh-dmg-rep.pdf```.
5. Press the blue **Send** button.
This API call does not return a value. Instead what you're looking for is the response code (200 OK). This code signifies that the request is successful and the API server has responded as required.


**Tell the Context API to Process the Uploaded File:**
At this step of the process you have uploaded a file to the S3 storage bucket using a presigned URL. The context API does not automatically trigger processing of this file, since it does not yet know what actions you want to include in your processing (remember the available actions from before). Here, we'll ask the Context API to process the uploaded file and include actions we want executed. 
The API expects the following:
- Bearer Token
- A JSON string containing two arrays: 
  - **objectKeys:** This value was returned when we requested the Presigned URL. We need to grab that value from that request and bring it here.
  - **actions:** A list of actions you want the Context API to execute on the file affecting the data returned. The actions must exactly match the actions received from the _GET Available Actions_ API call and should be in alignment with the document you uploaded. _I.E.: You wouldn't use ```image-description``` on a PDF document -or- text-based actioins on an image_.

1. From the list of collections, select the request titled ```POST Content processing```. 
2. From the tabs located below the HTTP location field, select **Body**.
3. You should still have the Request tab from when we requested the Presigned URL. _Select that tab._
4. In the results of the Presigned URL, copy the **objectKey value only**, leaving out the quotes. The value you copy will look like this:
```
contents/4c9b66de-fc0b-4c43-8fb1-40efb689bf72/d6aac838-ed99-4693-8833-cd9eabe008be/d6aac838-ed99-4693-8833-cd9eabe008b
```
5. Reselect the ```POST Content processing``` Request tab. 
6. Paste the _objectKey_ into the JSON string as a value to "objectKeys", replacing the text this is there. Be sure to paste the value between the quotation marks and not paste over them. 
7. Supply the actions by pasting the following text between the hard brackets [] as the value to "actions".
```
"text-summarization","named-entity-recognition-text"
```
Your entire JSON string should look something like this:
```
{
 "objectKeys": ["contents/4c9b66de-fc0b-4c43-8fb1-40efb689bf72/d6aac838-ed99-4693-8833-cd9eabe008be/d6aac838-ed99-4693-8833-cd9eabe008b"],
 "actions": ["text-summarization","named-entity-recognition-text"]
}
```
8. Press the blue **Send** button.
This API call will return a _processingID_ in a JSON string. This value will be used to get the results of the job. Your return should look like this:
```
{
    "processingId": "9afbda37-c512-4996-9141-46783cb4ar676"
}
```


**Get the Results from the Document Curation:**
Finally, we'll send an HTTP request to the Context API and ask for the results from the job we processed.
The API expects the following:
- Bearer Token
- A processingID

1. From the list of collections, select the request titled ```GET Retrieve result by ID```. 
2. Press the blue **Send** button.
**NOTE:** If you process this request too soon after processing the ```POST Content processing``` request you may get a JSON reply stating that the job is still processing. It will look like this:
```
{
    "id": "b6148d17-1f47-4acb-b3ef-7fb55711fc06",
    "timestamp": "2025-09-25T18:43:07.5438266+00:00",
    "status": "PROCESSING",
    "inProgress": true
}
```
Once your ```POST Content processing``` has had time to process you should get a JSON reponse with all of the data you requested in the processing request. 

---

### Send a Image for Summarization & Named Entity Recognition:

**Get a Presigned URL**
This is essentially the same Request as before, but we'll change the document mime type for jpeg images.
1. You should already have a Request tab opened for the ```POST Authorize``` Request. _Select that tab_.
2. Press the blue **Send** button to get a new Bearer Token (This will refresh our Bearer Token since it expires).
3. From the list of collections, select the request titled ```GET PresignedURL```. 
4. In the address bar, the API address ends with the text "application%2Fpdf". Change only that text to: ```image%2Fjpg```.
5. Press the blue **Send** button.
The resulting presigned URL is ready to receive a jpeg image.


**Upload an Image to the S3 Bucket**
1. From the list of collections, select the request titled ```PUT Upload File```. 
2. From the tabs located below the HTTP location field, select **Body**.
3. Click on the empty **Select file** field to open a file prompt.
4. Navigate to this path from the root level of the files provided: ```/Files to Upload/``` and select any one of the jpeg images you find there.
5. Press the blue **Send** button.
This API call does not return a value. Instead what you're looking for is the response code (200 OK). This code signifies that the request is successful and the API server has responded as required.


**Tell the Context API to Process the Uploaded Image:**
1. From the list of collections, select the request titled ```POST Content processing```. 
2. From the tabs located below the HTTP location field, select **Body**.
3. You should still have the Request tab from when we requested the Presigned URL. _Select that tab._
4. In the results of the Presigned URL, copy the **objectKey value only**, leaving out the quotes. The value you copy will look like this:
```
contents/4c9b66de-fc0b-4c43-8fb1-40efb689bf72/d6aac838-ed99-4693-8833-cd9eabe008be/d6aac838-ed99-4693-8833-cd9eabe008b
```
5. Reselect the ```POST Content processing``` Request tab. 
6. Paste the _objectKey_ into the JSON string as a value to "objectKeys", replacing the text this is there. Be sure to paste the value between the quotation marks and not paste over them. 
7. Supply the actions by pasting the following text between the hard brackets [] as the value to "actions".
```
"image-description","named-entity-recognition-image"
```
Your entire JSON string should look something like this:
```
{
 "objectKeys": ["contents/4c9b66de-fc0b-4c43-8fb1-40efb689bf72/d6aac838-ed99-4693-8833-cd9eabe008be/d6aac838-ed99-4693-8833-cd9eabe008b"],
 "actions": ["image-description","named-entity-recognition-image"]
}
```
8. Press the blue **Send** button.
This API call will return a _processingID_ in a JSON string. This value will be used to get the results of the job. Your return should look like this:
```
{
    "processingId": "9afbda37-c512-4996-9141-46783cb4ar676"
}
```


**Get the Results from the Image Curation:**
1. From the list of collections, select the request titled ```GET Retrieve result by ID```. 
2. Press the blue **Send** button.
**NOTE:** If you process this request too soon after processing the ```POST Content processing``` request you may get a JSON reply stating that the job is still processing. It will look like this:
```
{
    "id": "b6148d17-1f47-4acb-b3ef-7fb55711fc06",
    "timestamp": "2025-09-25T18:43:07.5438266+00:00",
    "status": "PROCESSING",
    "inProgress": true
}
```
Once your ```POST Content processing``` has had time to process you should get a JSON reponse with all of the data you requested in the processing request. 


---




