# Diligent SpeedyCheers

## Functional Requirements

- Request Submission: Users must have the capability to submit a "Request a Cheer" through the application, detailing their current location and possibly the type of cheer they need.

- partner Notification: The nearest SpeedyCheer partner to the user’s location must be automatically notified of the new cheer request, including the location and details of the request.

- Accepting Requests: partners should be able to accept cheer requests through their version of the application. Upon acceptance, the user is notified that a partner is on their way.

- Session Timing and Billing: The application must track the duration of each cheer session, starting from the partner's arrival to the completion of the task. Users are billed in 5-minute increments at a rate of $10 per 5 minutes.

- Location Services: Real-time location tracking for partners to efficiently match them with nearby user requests. 

- Optional - Online Payment Services

## Architeture Diagram

![SpeedyCheers Architecture](./speedycheersArch.png)


## Main Flows

### User Authentication Flow:

Users and partners authenticate via the Client UI, which interacts with Amazon Cognito.
Reason: Amazon Cognito provides secure user sign-up, sign-in, and access control, offloading this complexity from the application.

### Cheer Request Submission Flow:

Authenticated users submit cheer requests through the Client UI.
The request hits API Gateway, which invokes a Lambda `CheerRequestHandler`.
This function validates the request, saves it to DynamoDB, and pushes a message into the `CheerRequestQueue`.
Reason: API Gateway and Lambda provide a serverless, scalable front-end that can handle variable traffic patterns. DynamoDB offers fast, scalable NoSQL storage. SQS decouples the request handling from the processing, enhancing system reliability.

### Match Finding Flow:

`MatchingServiceWorker` polls the `CheerRequestQueue` and processes the messages.
If a match is not found, the request is sent to the `NoMatchDLQ` (Dead Letter Queue) for further inspection and handling.
If a match is found, the function publishes a message to an SNS topic that then sends the match notification to the `MatchFoundQueue`.
Reason: Using DLQs ensures that unprocessed messages are not lost and can be audited or reprocessed. SNS facilitates a pub/sub messaging pattern, ideal for broadcasting messages like found matches.

### Partner Notification Flow:

A Lambda function `NotificationHandler` listens to the `MatchFoundQueue` and sends real-time notifications to the appropriate partner through a WebSocket connection managed by API Gateway.
Reason: Real-time WebSocket connections are necessary for partners to receive immediate notifications. API Gateway's management of these connections offloads the complexity and provides scalability.


## API Signatures
Given the main flows, the following API signatures would be critical for the system:

 - POST /login: Authenticate users and partners
 - POST /cheerRequest: Submit a cheer request
 - GET /cheerRequest/{requestId}: Retrieve the status of a cheer request (used for polling)
 - POST /updatePaymentInfo: Send the payment status from 3rd party to payments table
 - POST /updateLocation : update the location to 
 - WebSocket /partner/notification: partner connects to the API Gateway websocket api
 - POST /rideAccept : When partner accepts the ride
 - POST /rideStart : When partner starts cheering
 - POST /rideEnd : When partner ends cheering

## Database Schema


### CheerRequestTable

| Attribute Name | Data Type | Primary Key | Indexes                               |
|----------------|-----------|-------------|---------------------------------------|
| RequestID      | String    | Primary Key | UserID-index (Global Secondary Index) |
| UserID         | String    |             |                                       |
| Location       | Map       |             |                                       |
| RequestType    | String    |             |                                       |
| Status         | String    |             |                                       |
| CreatedAt      | Number    |             |                                       |

### RideTable

| Attribute Name | Data Type | Primary Key | Indexes                                          |
|----------------|-----------|-------------|--------------------------------------------------|
| RideID         | String    | Primary Key | partnerID-index (Global Secondary Index)          |
| RequestID      | String    |             | RequestID-index (Global Secondary Index)         |
| partnerID       | String    |             |                                                  |
| StartTime      | Number    |             |                                                  |
| EndTime        | Number    |             |                                                  |
| Status         | String    |             |                                                  |
| Fare           | Number    |             |                                                  |

### PartnerTable

| Attribute Name   | Data Type | Primary Key | Indexes                                      |
|------------------|-----------|-------------|----------------------------------------------|
| partnerID         | String    | Primary Key | Email-index (Global Secondary Index)         |
| Name             | String    |             | Status-index (Global Secondary Index)        |
| Type             | String    |             |                                              |
| Phone            | String    |             |                                              |
| CurrentLocation  | Map       |             |                                              |
| Status           | String    |             |                                              |

### PaymentsTable

| Attribute Name | Data Type | Primary Key | Indexes                                      |
|----------------|-----------|-------------|----------------------------------------------|
| PaymentID      | String    | Primary Key | RideID-index (Global Secondary Index)         |
| RideID         | String    |             | UserID-index (Global Secondary Index)         |
| UserID         | String    |             |                                              |
| Amount         | Number    |             |                                              |
| Status         | String    |             |                                              |
| ThirdPartyReferenceID    | String    |             |                                              |





## Future Scope for Improvement
 - Upgrade client polling to socket based notification:
 - Handle failover scenarios 
 - Real-time Monitoring and Analytics : AWS CloudWatch, Athena 
 - Auto-scaling strategy for Lambda functions and DynamoDB to handle peak loads
 - Implement machine learning models to predict cheer request patterns and position partners based on demand forecasting.
 - Integration with map service to show the real time partner position to client
 - Handling DLQ
 - Better implementation of Payments flow
 - Geo Location based deployment of cloud native service for reduced latency and faster processing. In this case since the transactions are mutually exclusive with respect to the cities. Geo location based routing will significantly boost the performance. 



