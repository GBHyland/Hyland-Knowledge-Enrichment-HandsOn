# Content Innovation Cloud - Configuration for Knowledge Enrichment
This guide is intended to walk you through the steps to configure necessary elements in Hyland's Content Innovation Cloud to support Knowledge Enrichment. _To complete these steps, you will:_
- Create a Service user
- Create an External Application

---

### Create a Service User
1. Navigate to the [Content Innovation Cloud platform](https://www.experience.hyland.com/) in your browser and sign in with your provided credentials.
2. Select **Administration Portal**.
3. Select **Identity** from the top navigation banner.
4. From the left panel, toggle down **Users**.
5. Select **Service Users** from the left panel under Users.
6. Select the purple **Create Service User** button located near the top right of the page.
7. In the popup creation wizard, enter the following values:
   1. **Username:** ```cic-ke-class``` - _Press Next Step_
   2. **User Group:** add ```BB Context Enrichment Users``` and ```BB Data Curation Users``` - _Press Next Step_
   3. **Application Name:** ```cic-ke-class-app``` - _Press Next Step_
   4. Review the information for accuracy, then press the **Create Service User** button.
   5. Copy both the **client_id** and **client_secret** from the Summary page **before** you close this window. Store this information in a text file on your local machine for use later. _Press the **Close** button_

The above steps created both the Service User and the External Application needed to Authenticate with Knowledge Anrichment APIs, however some further configuraiton is necessary for the External Application. _Continue to the next section._

<br>

### Configure the External Application
1. In the Admin Portal, select **External Systems** from the top navigatioin banner.
2. In the _External Systems_ list, find and select the application you created above, titled: ```cic-ke-class-app```.
3. Toggle down the **Allowed Scoped** drop-down field.
4. Deselct **ALL** of the default selected items.
5. Select only **Environment Authorization** and click away from the drop-down menu to close it.
6. A new drop-down field appears called **Environment**. Select **DEV** from this drop-down.
7. Toggle down the **Application** drop-down field and select ```Data Curation API``` from the options.
8. Press the **Save** button from the bottom-right corner of the page.

---

### Knowledge Enrichment API
After completing the steps above, you're ready to engage with the API.
[Knowledge Enrichment API Guide](./ke-api-guide.md)

---
