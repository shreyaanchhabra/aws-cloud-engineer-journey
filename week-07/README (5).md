# Week 7: Serverless & Event-Driven Architecture

## How a request flows

```
Client (curl / browser)
   |
   |  GET /properties/R100234
   v
API Gateway  (HTTP API, route: GET /properties/{id})
   |
   |  passes the {id} part of the URL to Lambda as event["pathParameters"]["id"]
   v
Lambda  get-property  (Python 3.12)
   |
   |  table.get_item(Key={"L_ListingID": "R100234"})
   |  execution role only allows dynamodb:GetItem on table/Properties
   v
DynamoDB  Properties  (partition key: L_ListingID)
   |
   v
Item found  -> 200 + listing JSON
Not found   -> 404 {"error": "not found"}
```

## Summary

In Week 3, PropertyLite ran on an EC2 instance that stayed on all the time, even when nobody was using it. This week I moved its read path, looking up one listing by ID, onto serverless services so it only runs when a request comes in. API Gateway gives me a public URL with the route `GET /properties/{id}` and hands each request to a Lambda function. The Lambda (`lambda_function.py`) does the same job as PropertyLite's `get_property` route, except it reads the listing from a DynamoDB table called `Properties` instead of the CSV file. When I called `/properties/R100234`, I got back the same Sacramento listing with the same fields that the EC2 version returned, and an ID that doesn't exist gives a 404. The test calls I used are in `curl-tests.sh`.

For permissions, I added one inline policy to the Lambda's execution role that allows only `dynamodb:GetItem` on `arn:aws:dynamodb:*:*:table/Properties`. The function can look up a single item in that one table, and nothing else, so it can't write or delete data or reach any other table or service. It's the same least-privilege approach I used for the S3 policy in Week 2. If an API call ever fails with a 500, the response alone doesn't explain why, so the first step is checking the function's CloudWatch logs, which show the real error, like a permissions problem or a missing path parameter.

Last, I created an SQS queue called `property-queue` with a dead-letter queue, `property-dlq`, and set `maxReceiveCount` to 3. I sent a test message and read it back from the queue. If a consumer fails to process a message three times, SQS moves it to the dead-letter queue, where it can be inspected instead of being retried forever or lost.
