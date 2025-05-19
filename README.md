# Leveraging Prompt Templates Across Salesforce Ecosystem

## Overview

Salesforce’s **Prompt Builder** empowers organizations to integrate generative AI seamlessly into business processes by dynamically generating context-aware responses. Whether automating case summaries, drafting personalized emails, or enriching field data, Prompt Templates can be invoked through multiple entry points—**Flow, Apex, or REST API**—enabling AI-driven automation across declarative and programmatic solutions.

This guide explores how to integrate Prompt Templates into Salesforce workflows, backend logic, and external systems, with a focus on a **Case Classification & Summarization** use case.

## What is Prompt Builder?

Prompt Builder, part of **Einstein 1 Studio**, enables integration of generative AI into everyday workflows by allowing you to build and manage prompt templates. These templates can dynamically merge Salesforce CRM data—record fields, flows, Apex inputs, related lists—into context-aware prompts for Large Language Models (LLMs).

---

## Entry points for executing prompt templates

![Prompt life cycle](https://github.com/user-attachments/assets/d0a733a1-21e5-4ffa-854e-9a72f1e1b7b8)


## Entry Point Matrix

| Template Type        | Description                                                | Inputs                                      | Available Entry Points                               |
| -------------------- | ---------------------------------------------------------- | ------------------------------------------- | ---------------------------------------------------- |
| **Sales Email**      | Draft personalized emails through Email Composer           | Contact or Lead + optional secondary object | Email Composer, Flow, Apex, REST API, Copilot action |
| **Field Generation** | Generate text for storage in custom fields on record pages | Single object of choice                     | Record Page, Flow, Apex, REST API, Copilot action    |
| **Flex**             | Generate text for flexible use cases                       | Up to five objects of choice                | Flow, Apex, REST API, Copilot action                 |
| **Record Summary**   | Generate record summaries for Einstein Copilot             | Single object of choice                     | Copilot action                                       |

---

## Entry Points Reference

| Out-of-the-Box Entry Points | Custom Entry Points              |
| --------------------------- | -------------------------------- |
| Email Composer (OOTB)       | Flow (Declarative)               |
| Dynamic Record Pages (OOTB) | Apex (Backend logic)             |
| Copilot Actions (OOTB)      | REST API (External Applications) |

---

## Execution Flow

1. **Invocation**: Triggered via Flow, Apex, REST API, or Copilot
2. **Transformation**: Resolves data (fields, merge references, etc.) via Trust Layer
3. **LLM Processing**: Prompt sent to LLM for generation
4. **Response Handling**: Output delivered to calling system for further use

---

## Best Practices

* Use REST API for integrations with external systems
* Use Apex for complex logic
* Use Flow for declarative automation
* Use Copilot for user-facing AI actions

---

# Use Case: Case Classification & Summarization System

## System Overview

This system classifies and summarizes service cases (mechanical, electrical, electronic, structural) using the AI capabilities of Prompt Builder integrated through Flow, Apex, and REST API.

### Prompt Template Details

* **Name:** Summarize\_Classify\_Cases1
* **Function:** Classify `Type`, `Reason` and generate `Quick_Summary__c`

#### Sample Prompt Template

```
You are a API service for a company that services mechanical, electrical, electronic, and other forms of hardware. Your job is to categorize cases by Type and Case Reason and summarize them using the Case's Case Comments, Subject, and Description.
Case Type and Case Reason have predefined values. They are listed below:
Case Type: Electrical, Mechanical, Electronic, Structural, Other, null
Case Reason: Installation, Equipment Complexity, Performance, Breakdown, Equipment Design, Feedback, Other, null
Categorize the case below.
Priority: {!$Input:myCase.Priority}
Subject: {!$Input:myCase.Subject}
Description: {!$Input:myCase.Description}
Case Comments: {!$RelatedList:mycase.CaseComments.Records}
Now categorize and summarize the case. Output your categorization and summary as valid JSON with the keys "caseType", "reason", and "summary". Always include each key.Ensure that you return valid JSON.

Below is an example output:
{
"caseType": "Mechanical",
"reason": "Installation",
"summary": "The case titled 'Mechanical issue' has a priority level of High and is categorized under Mechanical. The case comments indicate that engineering was contacted to obtain a PDF of the installation instructions. The diagram was then emailed to the customer, and a response is currently awaited. Additional information shared notes that the device was manufactured in 2020."
}
```

---

# Integration Methods

# A. Flow Integration

Every saved and activated prompt template that you create is automatically exposed as an invocable action that you can invoke from Flow using the regular Actions element. You can filter them out by selecting the Prompt Template category when creating the element.

All templates with a specific input should have that input passed in as a related entity when invoked from a flow.
In this example, since this is a field generation prompt template that’s associated with the Contact object, you need to pass a contact as a related entity.

* **Type:** Record-Triggered Flow
* **Trigger:** On Case Create/Update
* **Logic:**

  * Checks if `Reason`, `Type`, or `Quick_Summary__c` is blank
  * Invokes the prompt template using Flow action
  * Parses and updates Case using Apex class `CaseSummarizationTemplateParser`


## Demo :
 
 ![ezgif com-video-to-gif-converter](https://github.com/user-attachments/assets/859c8e81-fc22-4580-a40f-dcc4e9792728)


# B. APEX INTEGRATION

## Purpose

Use Apex to invoke Prompt Builder templates programmatically with full control over logic, error handling, and field updates.

## Key Components

* `ConnectApi.EinsteinLLM.generateMessagesForPromptTemplate(...)`: Main method to call the template
* `ConnectApi.EinsteinPromptTemplateGenerationsInput`: Holds the input record(s), configuration, and application info
* `ConnectApi.WrappedValue`: Wraps Salesforce object references as input

## Apex Sample Code Explained

```apex
public with sharing class DynamicPromptTemplateHandler {
    public static String generatePromptFromTemplate(Id caseId) {
        try {
            // Step 1: Prepare the Case input
            Map<String, Object> caseMap = new Map<String, Object>{'id' => caseId};
            ConnectApi.WrappedValue caseValue = new ConnectApi.WrappedValue();
            caseValue.value = caseMap;

            Map<String, ConnectApi.WrappedValue> inputParams = new Map<String, ConnectApi.WrappedValue>{
                'Input:mycase' => caseValue
            };

            // Step 2: Set additional LLM configuration
            ConnectApi.EinsteinLlmAdditionalConfigInput config = new ConnectApi.EinsteinLlmAdditionalConfigInput();
            config.applicationName = 'PromptBuilderPreview'; // Logical grouping name
            config.temperature = 0.2; // Controls randomness in response
            config.numGenerations = 1; // Number of response outputs

            // Step 3: Create request payload
            ConnectApi.EinsteinPromptTemplateGenerationsInput request = new ConnectApi.EinsteinPromptTemplateGenerationsInput();
            request.additionalConfig = config;
            request.inputParams = inputParams;

            // Step 4: Call the Prompt Template
            ConnectApi.EinsteinPromptTemplateGenerationsRepresentation response =
                ConnectApi.EinsteinLLM.generateMessagesForPromptTemplate(
                    'Summarize_Classify_Cases1', // Template API name
                    request
                );

            // Step 5: Parse and update the result
            if (response != null && !response.generations.isEmpty()) {
                String summary = response.generations[0].text;
                Case c = [SELECT Id, Quick_Summary__c FROM Case WHERE Id = :caseId];
                c.Quick_Summary__c = summary;
                update c;
                return summary;
            }
            return null;
        } catch (Exception e) {
            System.debug('Error: ' + e.getMessage());
            return null;
        }
    }
}
```

### Highlights:

* Handles one record per request (`Input:mycase`)
* Use the Template API Name as defined in Prompt Builder
* Returns the generated text from the first response
* Suitable for calling inside Flows, Triggers, or Custom Buttons

## Demo : 

![ScreenRecording2025-05-19at10 09 04PM-ezgif com-speed (1)](https://github.com/user-attachments/assets/4ed8778c-4667-4eaf-999d-c04b865b074c)

---

# C. REST API INTEGRATION

## Purpose

Use the REST API to invoke Prompt Templates from external applications, systems, or integrations such as Node.js, Python, or middleware platforms like MuleSoft.

## Endpoint Format

```
POST /services/data/v62.0/einstein/prompt-templates/{TEMPLATE_API_NAME}/generations
```

## Example Request Body

```json
{
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
}
```

### Field Breakdown:

* `isPreview`: Set to `false` to trigger real LLM execution
* `inputParams.valueMap`: Provide the input object ID (e.g., Case ID)
* `additionalConfig`:

  * `numGenerations`: Number of completions
  * `temperature`: Lower means more deterministic
  * `applicationName`: Logical label for identification

## Sample cURL Command

```bash
curl --location --request POST 'https:mydomain.develop.my.salesforce.com/services/data/v62.0/einstein/prompt-templates/Template_Dev_Name/generations' \
--header 'Authorization: Bearer {access_token}' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--data-raw '{
  "isPreview": false,
  "inputParams": {
    "valueMap": {
      "Input:mycase(Prompt inputs)": {
        "value": {
          "id": " "
        }
      }
    }
  },
  "additionalConfig": {
    "numGenerations": 1,
    "temperature": 0,
    "frequencyPenalty": 0.0,
    "presencePenalty": 0.0,
    "additionalParameters": {},
    "applicationName": "PromptBuilderPreview"
  }
}'
```
## Demo : 

![ScreenRecording2025-05-19at10 22 50PM-ezgif com-speed](https://github.com/user-attachments/assets/e82cfb9b-1c4f-4783-92d2-4ae4d3459637)


### Use Cases

* Node.js/React front-end app invoking classification service
* Middleware performing summarization as part of ETL flow
* Third-party integration submitting records for AI processing

---

> ✅ **Tip:** Always test your Apex or REST request in Postman or Developer Console before moving to production.

> ✅ **Tip:** Use the same prompt API name and input mappings as configured in your Prompt Builder template.

---

## Resources

* [Apex ConnectApi Overview](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/connectAPI_overview.htm)
* [Prompt Template Connect REST Resource](https://developer.salesforce.com/docs/atlas.en-us.chatterapi.meta/chatterapi/connect_resources_prompt_template.htm)
* [Prompt Builder Documentation](https://help.salesforce.com/s/articleView?id=ai.prompt_builder_about.htm&type=5)

