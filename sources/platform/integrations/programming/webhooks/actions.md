---
title: Webhook actions
description: Send notifications when specific events occur in your Actor/task  run or build. Dynamically add data to the notification payload.
sidebar_position: 2
slug: /integrations/webhooks/actions
---

**Send notifications when specific events occur in your Actor/task  run or build. Dynamically add data to the notification payload.**

---

## Send HTTP request

To send notification, you can use the HTTP request action, which sends an HTTP POST request to a specified URL with a JSON payload. The payload is defined using a payload template, which is a JSON-like syntax that allows you to include variables enclosed in double curly braces `{{variable}}`. This enables the
dynamic injection of data into the payload when the webhook is triggered.

:::tip Webhook delivery

Webhooks are sent from servers that use static IP addresses. If your webhook destination is secured by a firewall, you can add the following IP addresses to your allow list to ensure Apify webhooks are delivered successfully.

<details>
    <summary>IP list</summary>
- `3.215.64.207`
- `13.216.80.7`
- `13.216.180.86`
- `34.224.107.31`
- `34.236.208.85`
- `44.198.219.104`
- `44.207.71.44`
- `44.207.141.205`
- `52.4.20.206`
- `52.203.255.236`
</details>

:::

### Response management

The response to the POST request must have an HTTP status code in the `2XX` range. If the response has a different status code, it is considered an error, and the request will be retried periodically with an exponential back-off:

- First retry: after approximately _1 minute_
- Second retry: after _2 minutes_
- Third retry: after _4 minutes_
- ...
- Eleventh retry: after approximately _32 hours_

If the request fails after _11 retries_, the system stops retrying.

### Security considerations

For security reasons, include a secret token in the webhook URL to ensure that only Apify can invoke it. You can use the **Test** button in the user interface to test your endpoint.

:::tip Headers template

You can also use [Headers template](/platform/integrations/webhooks/actions#headers-template) for this purpose.

:::

Note that webhook HTTP requests have a timeout of _30 seconds_.
If your endpoint performs a time-consuming operation, respond to the request immediately to prevent timeouts before Apify receives the response. To ensure reliable completion of the time-consuming operation, consider using a message queue internally to retry the operation on internal failure.

In rare cases, the webhook might be invoked more than once.
Design your code to be idempotent to handle duplicate calls.

:::note Apify requests: auto-added tokens

If the URL of your request points toward Apify, you don't need to add a token, since it will be added automatically.

:::

## Payload template

The payload template is a JSON-like string that allows you to define a custom payload structure and inject dynamic data known only at the time of the webhook's invocation. Apart from the variables, the string must be a valid JSON.

Variables must be enclosed in double curly braces and can only use the pre-defined variables listed in the [Available variables](#available-variables) section. Using any other variable will result in a validation error.

The syntax of a variable is: `{{oneOfAvailableVariables}}`. Variables support accessing nested properties using dot notation: `{{variable.property}}`.

### Default payload template

```json5
{
    "userId": {{userId}},
    "createdAt": {{createdAt}},
    "eventType": {{eventType}},
    "eventData": {{eventData}},
    "resource": {{resource}}
}
```

### Default payload example

```json5
{
    "userId": "abf6vtB2nvQZ4nJzo",
    "createdAt": "2019-01-09T15:59:56.408Z",
    "eventType": "ACTOR.RUN.SUCCEEDED",
    "eventData": {
        "actorId": "fW4MyDhgwtMLrB987",
        "actorRunId": "uPBN9qaKd2iLs5naZ"
    },
    "resource": {
        "id": "uPBN9qaKd2iLs5naZ",
        "actId": "fW4MyDhgwtMLrB987",
        "userId": "abf6vtB2nvQZ4nJzo",
        "startedAt": "2019-01-09T15:59:40.750Z",
        "finishedAt": "2019-01-09T15:59:56.408Z",
        "status": "SUCCEEDED",
        // ...
    }
}
```

#### String interpolation

The payload template is _not_ a valid JSON by default, but the resulting payload is. To use templates that provide the same functionality and are valid JSON at the same time, you can use string interpolation.

With string interpolation, the default payload template looks like this:

```json
{
    "userId": "{{userId}}",
    "createdAt": "{{createdAt}}",
    "eventType": "{{eventType}}",
    "eventData": "{{eventData}}",
    "resource": "{{resource}}"
}
```

If the string being interpolated contains only the variable, the actual variable value is used in the payload. For example `"{{eventData}}"` results in an object. If the string contains more than just the variable, the string value of the variable will appear in the payload:

```json5
{ "text": "My user id is {{userId}}" }
{ "text": "My user id is abf6vtB2nvQZ4nJzo" }
```

To enable string interpolation, use **Interpolate variables in string fields** switch within the Apify Console. In JS API Client it's called `shouldInterpolateStrings`. This field is always `true` when integrating Actors or tasks.

### Payload template example

This example shows how to use payload template variables to send a custom object that displays the status of a run, its ID and a custom property:

```json5
{
    "runId": {{resource.id}},
    "runStatus": {{resource.status}},
    "myProp": "hello world"
}
```

Note that the `eventData` and `resource` properties contain redundant data for backward compatibility. You can use either `eventData` or `resource` in your templates, depending on your use case.

## Headers template

The headers template is a JSON-like text where you can add additional information to the default HTTP header of the webhook request. You can pass the variables in the same way as in [payload template](#payload-template), including the use of string interpolation and the available variables. The resulting interpolated text need to be a valid JSON object, and values can be strings only.

Note that the following HTTP headers are always set by the system and your changes will always be rewritten:

| Variable                  | Value                   |
|---------------------------|-------------------------|
| `Host`                    | Request URL             |
| `Content-Type`            | `application/json`      |
| `X-Apify-Webhook`         | Apify internal value             |
| `X-Apify-Webhook-Dispatch-Id` | Apify webhook dispatch ID            |
| `X-Apify-Request-Origin`   | Apify origin           |

## Description

The description is an optional string that you can add to the webhook. It serves for your information and is not sent with the HTTP request when the webhook is dispatched.

## Available variables

| Variable    | Type   | Description                                                                         |
|-------------|--------|-------------------------------------------------------------------------------------|
| `userId`    | string | ID of the Apify user who owns the webhook.                                                |
| `createdAt` | string | ISO string date of the webhook's trigger event.                                     |
| `eventType` | string | Type of the trigger event, see [Events](/platform/integrations/webhooks/events).              |
| `eventData` | Object | Data associated with the trigger event, see [Events](/platform/integrations/webhooks/events). |
| `resource`  | Object | The resource that caused the trigger event.                 |
| `globals`   | Object | Data available in global context. Contains `dateISO` (date of webhook's trigger event in ISO 8601 format) and `dateUnix` (date of trigger event in Unix time in seconds) |

### Resource

The `resource` variable represents the triggering system resource. For example, when using the `ACTOR.RUN.SUCCEEDED` event, the resource is the Actor run. The variable will be replaced by the `Object` that you would receive as a response from the relevant API at the moment when the webhook is triggered. For the Actor run resource, it would be the response of the [Get Actor run](/api/v2/actor-run-get) API endpoint.

In addition to Actor runs, webhooks also support various events related to Actor builds. In such cases, the resource object will look like the response of the [Get Actor build](/api/v2/actor-build-get) API endpoint.
