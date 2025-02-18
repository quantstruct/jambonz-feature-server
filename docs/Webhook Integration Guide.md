
# Webhook Integration Guide

**Description:** This document provides comprehensive information about the asynchronous callbacks and webhooks used by the feature server. Webhooks are a core component of our platform, enabling real-time updates and event-driven interactions.

**Target Audience:** Developers, Integration Specialists

## Webhook Types

Our platform utilizes webhooks to notify your application about various events. Understanding the different webhook types is crucial for building robust integrations.  The following webhook types are currently supported:

*   **Call Update Webhooks:** Triggered when the status of a call changes (e.g., initiated, ringing, answered, completed).  These webhooks are sent to the URL specified when creating or updating a call.
*   **Conference Webhooks:**  Triggered for events related to conferences, such as participant joining, participant leaving, conference started, and conference ended.  These webhooks are sent to the URL specified when creating or updating a conference.
*   **Dequeue Webhooks:** Triggered when a call is dequeued from a queue.  These webhooks are sent to the URL specified when adding a call to the queue.

## Payload Reference

Each webhook sends a JSON payload containing information about the event that triggered it.  The structure of the payload varies depending on the webhook type.

### Call Update Webhook Payload

```json
{
  "event": "call.updated",
  "callSid": "CAxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "status": "completed",
  "startTime": "2023-10-27T10:00:00Z",
  "endTime": "2023-10-27T10:05:00Z",
  "duration": 300,
  "direction": "inbound",
  "from": "+15551234567",
  "to": "+15557654321",
  "price": "0.01",
  "priceUnit": "USD",
  "errorCode": null,
  "errorMessage": null
}
```

**Fields:**

*   `event`:  The type of event that triggered the webhook (e.g., `call.updated`).
*   `callSid`:  The unique identifier for the call.
*   `status`:  The current status of the call (e.g., `queued`, `ringing`, `in-progress`, `completed`, `busy`, `failed`, `no-answer`, `canceled`).
*   `startTime`:  The date and time the call started (ISO 8601 format).
*   `endTime`:  The date and time the call ended (ISO 8601 format).
*   `duration`:  The duration of the call in seconds.
*   `direction`:  The direction of the call (`inbound` or `outbound`).
*   `from`:  The phone number of the caller.
*   `to`:  The phone number of the recipient.
*   `price`:  The cost of the call.
*   `priceUnit`:  The currency of the call cost.
*   `errorCode`:  An error code if the call failed.
*   `errorMessage`:  An error message if the call failed.

### Conference Webhook Payload

```json
{
  "event": "conference.participant.joined",
  "conferenceSid": "CFxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "callSid": "CAxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "participantSid": "PAxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "startTime": "2023-10-27T10:02:00Z"
}
```

**Fields:**

*   `event`: The type of event that triggered the webhook (e.g., `conference.participant.joined`, `conference.participant.left`, `conference.ended`).
*   `conferenceSid`: The unique identifier for the conference.
*   `callSid`: The unique identifier for the participant's call leg within the conference.
*   `participantSid`: The unique identifier for the participant.
*   `startTime`: The date and time the participant joined the conference (ISO 8601 format).

### Dequeue Webhook Payload

```json
{
  "event": "queue.dequeued",
  "callSid": "CAxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "queueSid": "QUxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "timeDequeued": "2023-10-27T10:01:00Z",
  "waitDuration": 60
}
```

**Fields:**

*   `event`: The type of event that triggered the webhook (`queue.dequeued`).
*   `callSid`: The unique identifier for the call that was dequeued.
*   `queueSid`: The unique identifier for the queue the call was dequeued from.
*   `timeDequeued`: The date and time the call was dequeued (ISO 8601 format).
*   `waitDuration`: The amount of time the call waited in the queue, in seconds.

## Security Best Practices

Securing your webhook endpoints is crucial to prevent unauthorized access and manipulation of data.  We strongly recommend implementing the following security measures:

*   **HTTPS:**  Always use HTTPS for your webhook endpoints to encrypt the data transmitted between our platform and your application.
*   **Authentication:**  Implement a mechanism to verify that the webhook requests are originating from our platform.  We recommend using a shared secret that is included in the webhook request headers.  You can then verify this secret on your end.

    **Example (Python):**

    ```python
    import hashlib
    import hmac
    import os
    from flask import Flask, request, abort

    app = Flask(__name__)

    # Replace with your actual shared secret
    SHARED_SECRET = os.environ.get("SHARED_SECRET", "your_shared_secret")

    def verify_signature(request, shared_secret):
        signature = request.headers.get('X-Webhook-Signature')
        if not signature:
            return False

        message = request.get_data()
        expected_signature = hmac.new(
            shared_secret.encode('utf-8'),
            message,
            hashlib.sha256
        ).hexdigest()

        return hmac.compare_digest(signature, expected_signature)


    @app.route('/webhook', methods=['POST'])
    def webhook_handler():
        if not verify_signature(request, SHARED_SECRET):
            abort(403)  # Forbidden

        data = request.get_json()
        print(f"Received webhook data: {data}")
        return "OK", 200

    if __name__ == '__main__':
        app.run(debug=True)
    ```

    **Explanation:**

    1.  The code retrieves the `X-Webhook-Signature` header from the request.
    2.  It calculates the expected signature using the shared secret and the request body.
    3.  It compares the received signature with the expected signature using `hmac.compare_digest` to prevent timing attacks.
    4.  If the signatures don't match, the request is rejected with a 403 Forbidden error.

*   **IP Whitelisting (Optional):**  You can optionally whitelist our platform's IP addresses to further restrict access to your webhook endpoints.  Contact support for a list of our current IP addresses.
*   **Input Validation:**  Always validate the data received in the webhook payload to prevent injection attacks and other security vulnerabilities.

## Reliability Patterns

Webhooks are inherently asynchronous, and network issues can sometimes lead to delivery failures.  To ensure reliable webhook delivery, we recommend implementing the following patterns:

*   **Idempotency:**  Design your webhook handlers to be idempotent.  This means that processing the same webhook payload multiple times should have the same effect as processing it once.  Use the `callSid` or `conferenceSid` to identify and track processed events.
*   **Retry Mechanism:**  Implement a retry mechanism to handle failed webhook deliveries.  Our platform will attempt to redeliver webhooks that fail due to temporary network issues.  However, it's best to have your own retry logic in place as well.
*   **Dead Letter Queue (DLQ):**  Configure a dead letter queue to capture webhooks that fail to be delivered after multiple retries.  This allows you to investigate and resolve the underlying issues.
*   **Rate Limiting:** Be aware of rate limits on your webhook endpoints. If you exceed these limits, webhooks may be dropped. Monitor your webhook endpoint performance and adjust your processing logic as needed.

## Troubleshooting

If you are experiencing issues with webhooks, consider the following troubleshooting steps:

*   **Check Your Logs:**  Examine your application logs for any errors or exceptions related to webhook processing.
*   **Verify Your Endpoint:**  Ensure that your webhook endpoint is accessible and returns a 200 OK status code.  Use tools like `curl` or `Postman` to test your endpoint.
*   **Inspect Webhook Headers:**  Check the webhook request headers for any clues about the issue.  Pay attention to the `X-Webhook-Signature` header if you are using authentication.
*   **Review Your Code:**  Carefully review your webhook handling code for any logical errors or bugs.
*   **Contact Support:**  If you are still unable to resolve the issue, contact our support team for assistance.  Provide detailed information about the problem, including the webhook type, payload, and any relevant error messages.

## Related Endpoints

The following API endpoints are related to webhooks:

*   `/v1/updateCall/{callSid}`:  Used to update a call and specify the webhook URL.
*   `/v1/conference/{callSid}`:  Used to create or update a conference and specify the webhook URL.
*   `/v1/dequeue/{callSid}`: Used to dequeue a call from a queue.  The webhook URL is typically configured at the queue level.
