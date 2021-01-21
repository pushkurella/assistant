---

copyright:
  years: 2019, 2021
lastupdated: "2021-01-21"

subcollection: assistant

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:external: target="_blank" .external}
{:deprecated: .deprecated}
{:important: .important}
{:note: .note}
{:tip: .tip}
{:pre: .pre}
{:codeblock: .codeblock}
{:screen: .screen}
{:javascript: .ph data-hd-programlang='javascript'}
{:java: .ph data-hd-programlang='java'}
{:python: .ph data-hd-programlang='python'}
{:swift: .ph data-hd-programlang='swift'}
{:video: .video}

# Making a call after processing a message
{: #webhook-post}

Make a call to an external service or application every time a response is rendered by the assistant.
{: shortdesc}

A webhook is a mechanism that allows you to call out to an external program based on events in your program. You can add a postmessage webhook to your assistant if you want the webhook to be triggered before each message response is returned to the customer.

The postmessage webhook works with the v2 `/message` API only (stateless and stateful). For more information, see the [API reference](https://cloud.ibm.com/apidocs/assistant/assistant-v2#message).
{: important}

You can use a postmessage webhook to do the following types of things:

- When a skill's response contains a custom ID instead of text, pass the ID to an external database to retrieve a stored text response.

You can use this webhook in coordination with the premessage webhook. For example, if you used the premessage webhook to strip personally identifiable information from the customer's input, you can use the postmessage webhook to add it back. Or if you use the premessage webhook to translate the customer's input to the language of the skill, you can use the postmessage webhook to translate the response back into the customer's native language before is is returned. For more information, see [Making a call before processing a message](/docs/assistant?topic=assistant-webhook-pre).

If you want to perform a one-time action when certain conditions are met during a conversation, use a dialog webhook instead. For more information about the dialog webhook, see [Making a programmatic call from dialog](/docs/assistant?topic=assistant-dialog-webhooks).

![Plus or Premium plan only](images/plus.png) For environments where private endpoints are in use, keep in mind that a webhook sends traffic over the internet. For more information, see [Private network endpoints](/docs/assistant?topic=assistant-security#security-private-endpoints).
{: note}

## Defining the webhook
{: #webhook-post-create}

You can define one webhook URL to use for processing every message response before it is shared with the customer.

The programmatic call to the external service must meet these requirements:

- The call must be a POST HTTP request.
- The format of the request and response must be in JSON. For example: `Content-Type: application/json`.

Do not set up and test your webhook in a production environment where the assistant is deployed and is interacting with customers.
{: important}

Use an external service that can execute and return a result in less than 8 seconds. Otherwise, the customer will experience a lag in the conversational exchange.
{: tip}

To add the webhook details, complete the following steps:

1.  From your assistant, click the ![Overflow menu](images/kebab.png) icon, and then choose **Settings**.

1.  Click **Webhooks > Post-message webhook**.

    The postmessage webhook is enabled by default.

1.  Decide whether to return an error if the webhook call fails.

    When enabled, everything stops until the processing step is completed successfully.
    {: important}

    If you have a critical postprocessing step that must be taken before you want to allow the response to be sent to the customer, then keep this setting enabled.

    Take steps to test the process that you are calling on a regular basis so you will know if it's down, and can change this setting to prevent all of your message calls from failing.

    If you disable this setting, then the assistant ignores any errors it encounters and continues to process the response without taking the processing step.

    If the postprocessing step is helpful but not critical, consider disabling this setting.

1.  In the **URL** field, add the URL for the external application to which you want to send HTTP POST request callouts.

    For example, maybe you store your assistant's responses in a separate content management system. When the skill understands the input, the processed dialog node returns a unique ID that corresponds to a response in your CMS. To call a service that retrieves a response from your CMS for a given unique ID, specify the URL for your service instance.

    ```bash
    https://my-webhook.com/get_answer
    ```
    {: codeblock}

    You must specify a URL that uses the SSL protocol, so specify a URL that begins with `https`.

    You cannot use a webhook to call a {{site.data.keyword.openwhisk_short}} action that uses token-based Identity and Access Management (IAM) authentication. However, you can make a call to a secured {{site.data.keyword.openwhisk_short}} web action.
    {: important}

1.  In the **Secret** field, add a private key to pass with the request that can be used to authenticate with the external service.

    The key must be specified as a text string, such as `purple unicorn`. Maximum length is 1,024 characters. You cannot specify a context variable.

    It is the responsibility of the external service to check for and verify the secret. If the external service does not require a token, specify any string you want. You cannot leave this field empty.

1.  In the Headers section, add any headers that you want to pass to the service one at a time by clicking **Add header**.

    For example, if the external application that you call returns a response, it might be able to send a response in multiple different formats. The webhook requires that the response is formatted in JSON. The following table illustrates how to add a header that indicates that you want the resulting value to be returned in JSON format.

    <table>
    <caption>Header example</caption>
      <tr>
      <th>Header name</th>
      <th>Header value</th>
      </tr>
      <tr>
      <td>Content-Type</td>
      <td>application/json</td>
      </tr>
    </table>

    The service automatically sends an `Authorization` header with a JWT; you do not need to add one.

Your webhook details are saved automatically.

## Webhook security
{: #webhook-pre-security}

To authenticate the webhook request, verify the JSON Web Token (JWT) that is sent with the request. The webhook microservice automatically generates a JWT and sends it in the `Authorization` header with each webhook call. It is your responsibility to add code to the external service that verifies the JWT.

For example, if you specify `purple unicorn` in the **Secret** field, you might add code similar to this:

```javascript
const jwt = require('jsonwebtoken');
...
const token = request.headers.authentication; // grab the "Authentication" header
try {
  const decoded = jwt.verify(token, 'purple unicorn');
} catch(err) {
  // error thrown if token is invalid
}
```
{: codeblock}

## Troubleshooting the webhook
{: #webhook-post-ts}

The following error codes can help you track down the cause of issues you might encounter. If you have a web chat integration, for example, you will know that your webhook has an issue if every test message you submit returns a message such as, `There is an error with the message you just sent, but feel free to ask me something else.` If this message is displayed, use a REST API tool, such as cURL, to send a test `/message` API request, so you can see the error code and full message that is returned.

| Error code and message | Description |
|------------|-------------|
| 422 Webhook responded with invalid JSON body | The webhook's HTTP response body could not be parsed as JSON. |
| 422 Webhook responded with `[500]` status code | There's a problem with the external service you called. The code failed or the external server refused the request. |
| 500 Processor Exception : `[connections to all backends failing]` | An error occurred in the webhook microservice. It could not connect to backend services. |
{: caption="Error code details" caption-side="top"}

## Example request body
{: #webhook-post-request-body}

It is useful to know the format of the request postmessage webhook body so that your external code can process it.

The payload contains the response body that is returned by your assitant for the v2 `/message` (stateful and stateless) API call. The event name `message_processed` indicates that the request is generated by the postmessage webhook. For more information about the message request body, see the [API reference](https://cloud.ibm.com/apidocs/assistant/assistant-v2#message){: external}.

The following sample shows how a simple request body is formatted.

```json
{
 "event": {
    "name": "message_processed"
},
"options": {},
"payload": {
    "output": {
        "intents": [
            {
                "intent": "General_Greetings",
                "confidence": 1
            }
        ],
        "entities": [],
        "generic": [
            {
                "response_type": "text",
                "text": "Hello. Good evening"
            }
        ]
    },
    "context": {
        "global": {
            "system": {
                "user_id": "test user",
                "turn_count": 11
            },
            "session_id": "sxxx"
        },
        "skills": {
            "main skill": {
                "user_defined": {
                    "var": "anthony"
                },
                "system": {
                    "state": "nnn"
                }
            }
        }
    }
}
```
{: codeblock}

## Example
{: #webhook-post-example}

This example shows you how to add `y'all` to the end of each response from the assistant.

In the postmessage webhook configuration page, the following values are specified:

- **URL**: https://us-south.functions.appdomain.cloud/api/v1/web/e97d2516-5ce4-4fd9-9d05-acc3dd8ennn/southernize/add_southern_charm
- **Secret**: none
- **Header name**: Content-Type
- **Header value**: application/json

The postmessage webhook calls an IBM Cloud Functions web action name `add_southern_charm`.

The node.js code in the `add_southern_charm` web action looks as follows:

```javascript
function main(params) {
	console.log(JSON.stringify(params))
  if (params.payload.output.generic[0].text !== '') {
      //Get the length of the input text
        var length = params.payload.output.generic[0].text.length;
        //create a substring that removes the last character from the input string, which is typically punctuation.
        var revision = params.payload.output.generic[0].text.substring(0,length-1);
        const response = {
            body : {
                payload : {
                    output : {
                        generic : [
                              {
                                  //Replace the input text with your shortened revision and append y'all to it.
                                "response_type": "text",
                                "text": revision + ', ' + 'y\'all.'
                              }
                        ],
                    },
                },
            },
        };
        return response;
  }
  else {
    return { 
        body : params
    }
  }
}
```
{: codeblock}

## Removing the webhook
{: #webhook-post-delete}

If you decide you do not want to process message responses, complete the following steps:

1.  From the assistant overview page, click the ![Overflow menu](images/kebab.png) icon, and then choose **Settings**.

1.  Click **Webhooks > Pre-message webhook**.

1.  Do one of the following things:

    - To change the webhook that you want to call, click **Delete webhook** to delete the currently specified URL and secret. You can then add a new URL and other details.
    - To stop calling a webhook to process every message response, click the *Post-message webhook* switch to disable the webhook altogether.