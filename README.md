**DynamoDB**

\- Created DynamoDB table.

\- Turned on DynamoDB stream details.



**Need an IAM role that lets the Lambda function write to DynamoDB \& log to CloudWatch Logs**

* OrderServiceLambdaRole



**Creating Lambda Function**

Attach IAM Role to Lambda Function (in the Change Default Execution Role - Use an existing role)

Added environment variable with key-value pair



Deployed Code in code tab.



Tested the code with Status code 201 expected.



Went into DynamoDB orders, explore table items and checked the orders were received or not.





**API Gateway**

-Created API

-Configured Stages, Routes



Invoke URL generate through API Gateway - https://szmhnuemfe.execute-api.us-east-2.amazonaws.com/prod





**Enable CORS (So the Web App Can Call the API)**



Went to CORS section in API Gateway

Configured allowed origins, allowed methods, allowed headers to all.



Went to deploy, stages and selected prod then deploy. (Order API deployed)



**Built the Frontend Order Submission App**



Create an index.html file, paste code for web app

Updated URL in code with actual Invoke URL



Went to web page , put order details, submit order, order created successfully message.



Went to DynamoDB -> Orders -> Explore Table Items

Final Test Order Check Completed



**Confirm DynamoDB Streams Are Enabled**





**Creating the process order function in Lambda**

* Added environment variable named orders table name with value as orders. It is a key-value pair.





\*\*\*\*\*\*\*\*\*\*\*\*

Every time Lambda is invoked, the Lambda handler will receive an event that roughly looks like ---

{

  "Records": \[

    {

      "eventName": "INSERT",

      "dynamodb": {

        "NewImage": {

          "orderId": { "S": "1234-5678" },

          "customerName": { "S": "Alice" },

          "amount": { "N": "1499" },

          "product": { "S": "Wireless Headphones" },

          "status": { "S": "PENDING" },

          "createdAt": { "S": "2025-12-01T10:30:00Z" }

        }

      }

    }

  ]

}

\*\*\*\*\*\*\*\*\*\*\*\*

**Added Code, Deployed to ProcessOrderFunction.**



**Connect the Table Stream to Lambda**

**-Go to Lambda console, add trigger with configuration from DynamoDB.**



Tested and Validated that Orders moved from Pending to Processed.



DynamoDB -> Tables -> Orders

Explore Table Items

The Order was processed✅





**Current WorkFlow** 



Frontend → API Gateway → CreateOrderFunction → DynamoDB (Orders)

DynamoDB Streams → ProcessOrderFunction → updates order in DynamoDB



We currently have no error handling, if in case an order fails to process.



We need retries, back-pressure handling, Dead-Letter Queues. For this, we will use Amazon SQS.



We will use DLQ to handle technical failures and retries.





**Amazon SQS**



Created a Standard Queue on SQS. This queue will work on collecting messages that repeatedly fail in the main queue. This is the Dead Letter Queue that will collect failed events.



Created another standard queue that is the main order processing queue, which is the primary queue that the Lambda function will read from.





**Update ProcessOrderFunction in Lambda to move the final decision of marking the order as processed to SQS Worker.**

* Add SQS Environment variable to ProcessOrderFunction in Lambda, set the VALUE of the variable to the URL of order processing queue.



Made changes to ProcessOrderFunction to accommodate SQS.



**Created OrderWorkerFunction in Lambda.**



* Added an environment variable.



* Added Trigger to OrderWorkerFunction from SQS.



* Added code to the function



**Tested two orders**

* Normal Amount - Succeeded, showed completed status.
* High Amount - Failed, showed failed status.



**Checked CloudWatch Logs, found order updates.**





**Checked SQS, Opened order-processing-dlq and found the SQS message regarding order that failed.**





**Event Routing with EventBridge Pipes**



Current Event Flow

* Frontend → API Gateway → CreateOrderFunction → DynamoDB (Orders)
* DynamoDB Streams → ProcessOrderFunction → SQS (order-processing-queue)
* OrderWorkerFunction consumes from SQS, updates order status
* Technical failures are retried and eventually land in a DLQ



**We need to learn how to route events based on conditions.** 

e.g: 

* High-value purchases go through fraud checks
* Large orders trigger priority fulfilment
* Certain product types go to dedicated workflows



**Next Step - Building an Event Routing Layer**

* Events flowing through the main SQS queue are evaluated
* Only high-value orders (e.g. amount > 10,000) are routed into a separate queue
* Events are filtered, transformed, and forwarded by EventBridge Pipes
* A dedicated Lambda function handles high-value events



**Different orders can follow different processing paths.**



**Create a high-value-orders-queue in SQS.**





**Creating the EventBridge Pipe**

* HighValueOrderPipe - It's source is SQS Queue, i.e. order-processing-queue.
* Creating a Clean and High-Value Payload through target input transformer.
* Fill code in Sample events/Event Payload, Transformer, Output Fields.



**Created the HighValueWorker Lambda Function**

* Attach trigger- trigger SQS
* Select high-value-orders-queue as queue
* Activate trigger



**Tested routing with two orders- normal and high value**



* CreateOrderFunction writes the order into DynamoDB.
* DynamoDB Streams triggers ProcessOrderFunction.
* ProcessOrderFunction sends the order to order-processing-queue and updates DynamoDB.





**Adding Observability**



* Created an SNS Topic for System Alerts
* Created a subscription - Selected Email Protocol - Entered my email id as endpoint





**Create CloudWatch alarm for DLQ Messages**



* Selected the SQS Metric - By Queue Name
* Chose DLQ as order-processing-dlq

Selected the metric : ApproximateNumberOfMessagesVisible



* Kept the period as 1 minute and condition as greater than or equal to 1



* Configured SNS Topic to order-processing-alerts and named the alarm as order-processing-alarm



**Updated the Lambda OrderWorkerFunction code to parse SQS message to get order details and the following:**



* Technical failure path (for teaching retries \& DLQ):



If notes contains "FAIL", it throws an error.

SQS retries the message; after 3 attempts, it lands in the DLQ.



* Business rule failure path:



If amount > 10000, it sets status = FAILED in DynamoDB and writes a failureReason.

No retry, because this is not a transient error.



* Success path:

For other orders, it sets status = COMPLETED and adds completedAt.



**Checked SQS Dead Letter Queue and CloudWatch alarms to confirm that DLQ Alarm has moved into ALARM state and SNS notification is delivered to email inbox.**





 

