# API Endpoints

This document describes the primary API endpoints consumed by, and potentially provided by, the Order Processing Application.

## 1. APIs Consumed by Order Processing Application

The Order Processing Application integrates with several external and internal services.

### 1.1. Payment Gateway API

* **Description:** External API for authorizing and capturing payments.
* **Provider:** Third-party payment gateway service.
* **Base URL:** `https://api.paymentgateway.com/v1` (Production)
* **Authentication:** OAuth 2.0 Client Credentials flow.
* **Endpoints:**
    * `POST /authorize`: Authorizes a payment amount against a customer's payment method.
        * **Request Body (JSON):**
            ```json
            {
              "transactionId": "string",
              "amount": "decimal",
              "currency": "string",
              "paymentMethodToken": "string",
              "customerReference": "string"
            }
            ```
        * **Response Body (JSON):**
            ```json
            {
              "transactionId": "string",
              "status": "AUTHORIZED" | "FAILED",
              "authorizationCode": "string",
              "message": "string"
            }
            ```
    * `POST /capture`: Captures a previously authorized payment.
        * **Request Body (JSON):**
            ```json
            {
              "transactionId": "string",
              "authorizationCode": "string",
              "amount": "decimal"
            }
            ```
        * **Response Body (JSON):**
            ```json
            {
              "transactionId": "string",
              "status": "CAPTURED" | "FAILED",
              "message": "string"
            }
            ```
* **Error Handling:** HTTP status codes (4xx for client errors, 5xx for server errors). Specific error codes documented by the payment gateway.

### 1.2. CRM Customer Lookup API

* **Description:** Internal API for retrieving customer details based on a customer ID.
* **Provider:** Internal CRM service.
* **Base URL:** `http://crm-service.internal/api/v2`
* **Authentication:** Internal API Key (passed in `X-API-KEY` header).
* **Endpoint:**
    * `GET /customers/{customerId}`: Retrieves customer details.
        * **Path Parameter:** `customerId` (string)
        * **Response Body (JSON):**
            ```json
            {
              "customerId": "string",
              "firstName": "string",
              "lastName": "string",
              "email": "string",
              "phone": "string"
            }
            ```
* **Error Handling:** Standard HTTP status codes.

---

## 2. APIs Provided by Order Processing Application

Currently, the Order Processing Application primarily consumes APIs and publishes messages to EMS queues. It does not directly expose any synchronous REST/SOAP APIs for external consumption.

### 2.1. Messaging APIs (via TIBCO EMS)

While not traditional HTTP/REST APIs, the application effectively provides "messaging APIs" for other services to consume its outputs.

* **Input Queue: `queue.order.new`**
    * **Description:** Expects new order requests from upstream systems.
    * **Message Format (JSON Schema):**
        ```json
        {
          "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
          "title": "NewOrderRequest",
          "type": "object",
          "properties": {
            "orderId": { "type": "string", "description": "Unique identifier for the order" },
            "customerId": { "type": "string", "description": "ID of the customer placing the order" },
            "items": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "productId": { "type": "string" },
                  "quantity": { "type": "integer" }
                },
                "required": ["productId", "quantity"]
              }
            },
            "totalAmount": { "type": "number" },
            "currency": { "type": "string" },
            "paymentDetails": { "type": "object" }
          },
          "required": ["orderId", "customerId", "items", "totalAmount"]
        }
        ```
    * **Consumption:** Expected to be consumed by the `OrderReceiver` BW process.

* **Output Queue: `queue.order.status`**
    * **Description:** Publishes real-time updates on the lifecycle status of an order.
    * **Message Format (JSON Schema):**
        ```json
        {
          "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
          "title": "OrderStatusUpdate",
          "type": "object",
          "properties": {
            "orderId": { "type": "string", "description": "Unique identifier for the order" },
            "status": {
              "type": "string",
              "enum": ["RECEIVED", "VALIDATED", "PAYMENT_AUTHORIZED", "ORDER_SAVED", "COMPLETED", "FAILED"],
              "description": "Current status of the order"
            },
            "timestamp": { "type": "string", "format": "date-time" },
            "message": { "type": "string", "description": "Optional: additional details about the status" }
          },
          "required": ["orderId", "status", "timestamp"]
        }
        ```
    * **Consumption:** Consumed by `Fulfillment Service` and `Notification Service`.