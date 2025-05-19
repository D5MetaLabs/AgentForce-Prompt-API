# Salesforce Case Classification & Summarization System

## Overview

This system automatically classifies and summarizes hardware service cases (mechanical, electrical, electronic, structural) using AI-powered prompt templates invoked through multiple integration methods.

Prompt templates are used to integrate generative AI capabilities into your applications and workflows. They are created in **Prompt Builder**, part of the **Einstein 1 Studio**. Upon invocation, prompt templates resolve all data references (merge fields, related lists, flow inputs, Apex inputs), then securely pass through the **Einstein Trust Layer**, and finally generate responses from the **large language model (LLM)**.

### 🤖 What is Prompt Builder?

Prompt Builder allows you to simplify your users’ daily tasks by integrating generative-AI moments into their workflow. With Prompt Builder, you can create, test, revise, customize, and manage prompt templates that incorporate your CRM data—merging record fields, flows, related lists, and Apex inputs—to deliver context-aware AI interactions.

Prompt templates can be triggered via:

* **Flow**
* **Apex**
* **REST API**

---

## System Components

### 1. Prompt Template Workspace

* **Template Name:** `Summarize_Classify_Cases1`
* **Purpose:** Categorize cases by Type and Reason, and generate concise summaries.

#### 🔹 Input Structure

```json
{
  "Priority": "{!$Input:myCase.Priority}",
  "Subject": "{!$Input:myCase.Subject}",
  "Description": "{!$Input:myCase.Description}",
  "CaseComments": "{!$RelatedList:mycase.CaseComments.Records}",
  "Snapshot": "{!$RecordSnapshot:mycase.Snapshot}"
}
```

#### 🔹 Output Format

```json
{
  "caseType": "Mechanical|Electrical|Electronic|Structural|Other",
  "reason": "Installation|Equipment Complexity|Performance|Breakdown|Equipment Design|Feedback|Other",
  "summary": "Generated case summary text"
}
```

---

### 2. Integration Methods

#### A. Flow Integration

* **Flow Type:** Record-Triggered Flow

* **Trigger:** On Case create/update

* **Logic:**

  * Checks if `Reason`, `Type`, or `Quick_Summary__c` is blank
  * Invokes prompt template with case context
  * Parses response via Apex `CaseSummarizationTemplateParser`
  * Updates fields on the Case record

* **Fields Used:** `Subject`, `Description`, `Reason`, `Type`, `Quick_Summary__c`

---

#### B. Apex Integration

First, like in Flow, each prompt template will expect different inputs to be passed in, so you need to create the inputs in Apex and pass them to the template. The sample flex prompt template is associated with a `Property__c` custom object, so we need to pass a property record when we invoke it from Apex.

In Apex, you use classes and methods from `ConnectApi` to invoke prompt templates. You can control several parameters of the invocation using the `EinsteinPromptTemplateGenerationsInput` object:

* `isPreview`: If `true`, only resolves the prompt and returns it without sending to the LLM.
* `numGenerations`: Number of responses to generate (default is 1).
* `temperature`: Controls model creativity (0 = deterministic, 1 = more diverse).
* `frequencyPenalty`: Reduces repetitiveness of generated tokens.

Here’s a working example:

```java
public with sharing class DynamicPromptTemplateHandler {
    public static String generatePromptFromTemplate(Id caseId) {
        try {
            Map<String, Object> caseMap = new Map<String, Object>{'id' => caseId};
            ConnectApi.WrappedValue caseValue = new ConnectApi.WrappedValue();
            caseValue.value = caseMap;

            Map<String, ConnectApi.WrappedValue> inputParams = new Map<String, ConnectApi.WrappedValue>{
                'Input:mycase' => caseValue
            };

            ConnectApi.EinsteinLlmAdditionalConfigInput config = new ConnectApi.EinsteinLlmAdditionalConfigInput();
            config.applicationName = 'PromptBuilderPreview';
            config.temperature = 0.2;
            config.frequencyPenalty = 0.0;
            config.numGenerations = 1;

            ConnectApi.EinsteinPromptTemplateGenerationsInput request = new ConnectApi.EinsteinPromptTemplateGenerationsInput();
            request.additionalConfig = config;
            request.isPreview = false;
            request.inputParams = inputParams;

            ConnectApi.EinsteinPromptTemplateGenerationsRepresentation response = 
                ConnectApi.EinsteinLLM.generateMessagesForPromptTemplate(
                    'Summarize_Classify_Cases1', 
                    request
                );

            if(response != null && !response.generations.isEmpty()) {
                String summary = response.generations[0].text;
                Case c = [SELECT Id, Quick_Summary__c FROM Case WHERE Id = :caseId];
                c.Quick_Summary__c = summary;
                update c;
                return summary;
            }
            return null;
        } catch(Exception e) {
            System.debug('Error: ' + e.getMessage());
            return null;
        }
    }
}
```

* **Note:** The response also includes a `safetyScoreRepresentation`, which evaluates response safety using a toxicity detection model.
* If your prompt template returns **valid JSON**, you can parse the output easily in Apex or JavaScript (useful for LWC).

---

#### C. REST API Integration

* **Endpoint:**
  `POST /services/data/v62.0/einstein/prompt-templates/Summarize_Classify_Cases1/generations`

* **Sample Request:**

```bash
curl --location 'https://yourinstance.salesforce.com/services/data/v62.0/einstein/prompt-templates/Summarize_Classify_Cases1/generations' \
--header 'Authorization: Bearer {access_token}' \
--header 'Content-Type: application/json' \
--data-raw '{
  "isPreview": false,
  "inputParams": {
    "valueMap": {
      "Input:mycase": {
        "value": {
          "id": "500gL000005ZwO1QAK"
        }
      }
    }
  },
  "additionalConfig": {
    "numGenerations": 1,
    "temperature": 0,
    "applicationName": "PromptBuilderPreview"
  }
}'
```

---

## 3. Field Reference

| Object      | Field               | Purpose                                 |
| ----------- | ------------------- | --------------------------------------- |
| Case        | Type                | AI classification output                |
| Case        | Reason              | AI reason classification                |
| Case        | Quick\_Summary\_\_c | AI-generated summary text               |
| Case        | Subject             | Input to the prompt                     |
| Case        | Description         | Input to the prompt                     |
| CaseComment | (Related List)      | Contextual input for enhanced summaries |

---

## Prompt Template Entry Points

Prompt templates can be executed through multiple entry points:

| Prompt Template Type | Description                       | Inputs                              | Entry Points                                         |
| -------------------- | --------------------------------- | ----------------------------------- | ---------------------------------------------------- |
| Sales Email          | Draft personalized emails         | Contact or Lead (+ optional object) | Email Composer, Flow, Apex, REST API, Copilot action |
| Field Generation     | Generate and store field value    | Any object                          | Record page, Flow, Apex, REST API, Copilot action    |
| Flex                 | Flexible, multi-object generation | Up to 5 objects                     | Flow, Apex, REST API, Copilot action                 |
| Record Summary       | Summary for Einstein Copilot      | Any object                          | Copilot action                                       |

You can invoke prompt templates using built-in entry points like Email Composer or Dynamic Forms. You can also create **custom entry points** using Flow, Apex, or REST APIs to embed AI into your Salesforce business logic and external applications.

---

## Implementation Notes

* ✅ **Response Format:** Expects JSON with keys `caseType`, `reason`, and `summary`
* ⚠️ **Error Handling:** Validate API success, null checks, and exceptions
* 🔐 **Security:** Use scoped OAuth tokens for REST API authentication
* 🧪 **Testing:** Test with varied real-life case inputs
* 📊 **Monitoring:** Review classification accuracy and summary clarity

---

## Example Output

```json
{
  "caseType": "Mechanical",
  "reason": "Installation",
  "summary": "The case involves installation issues with a mechanical component. Customer reports difficulty assembling parts according to the manual. Case comments indicate technical drawings have been sent for clarification."
}
```

---

## 🔗 Resources

* [Salesforce Prompt Builder Documentation](https://help.salesforce.com/)
* [Einstein 1 Studio Overview](https://www.salesforce.com/)
* [ConnectApi.EinsteinLLM Apex Reference](https://developer.salesforce.com/docs/)

> 💡 Be sure to also read our previous blog post on prompt templates to explore grounding techniques and best practices.
