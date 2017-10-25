footer: © NodeProgram.com, Node.University and Azat Mardan 2017
theme: Simple, 1
build-lists: true

[.slidenumbers: false] 
[.hide-footer]

# Serverless with AWS and Node

## How you can build scalable APIs in the cloud without virtual machines

![inline 100%](images/azat.jpeg)
Azat Mardan @azat_co

![inline right](images/nu.png)

---

# Meet Your Presenter: Azat Mardan

* Author of 14 books and over 20 online courses, taught over 500 engineers in-person and over 25,000 online 
* Founder of Node University
* Likes FinTech, blockchain and his coffee :coffee: with coconut oil

---

![fit](images/azats-books-covers.png)

^Works at a small finTech Company you probably never heard about

---

## Works as Capital One (in top 10 US banks) in Technology Fellows

![inline](images/c1-commercial.gif)

---

## My Experience in Software Engineering and IT/DevOps

* App / Software Developer who sends code to IT
* DevOps at a small startup: AWS, Joyent
* Prototype: Heroku, Parse, Firebase
* Enterprise cloud: AWS, Azure 

---

# We are going to use three Amazon Web Services:

* DynamoDB
* Lambda
* API Gateway

---

#  The tutorial is broken down in the following steps:

1. Create DynamoDB table
1. Create IAM role to access DynamoDB
1. Create AWS Lambda
1. Create API Gateway
1. Test
1. Clean up

---

The source code including highly useful bash scripts to create RESTful endpoints in API Gateway are in [the GitHub repository for the AWS Intermediate course](https://github.com/azat-co/aws-intermediate/tree/master/code/serverless).

---

# 1. Create DynamoDB table

---

## Access to AWS CLI

Before starting, make sure you have AWS CLI installed. If you don't know how to do it, then follow instruction in my beginner post on AWS CLI called [AWS CLI Tutorial: Creating a Web Server](https://node.university/blog/1077866/aws-cli). You also need to configure 🔧 the AWS CLI with the proper [access key and secret&nbsp;🔑](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) which you can copy from your web console and enter using `aws configure`. 


---

## Create a table in AWS DynamoDB

```sh
aws dynamodb create-table --table-name messages \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

---

We'll get back the Arn identifier which is like a unique resource ID in AWS:

```js
{
    "TableDescription": {
        "TableArn": "arn:aws:dynamodb:us-west-1:161599702702:table/messages",
        "AttributeDefinitions": [
            {
                "AttributeName": "id",
                "AttributeType": "N"
            }
        ],
        "ProvisionedThroughput": {
            "NumberOfDecreasesToday": 0,
            "WriteCapacityUnits": 5,
            "ReadCapacityUnits": 5
        },
        "TableSizeBytes": 0,
        "TableName": "messages",
        "TableStatus": "CREATING",
        "KeySchema": [
            {
                "KeyType": "HASH",
                "AttributeName": "id"
            }
        ],
        "ItemCount": 0,
        "CreationDateTime": 1493219395.933
    }
}
```


---

We can also get this info by running another AWS CLI command:

```sh
aws dynamodb describe-table --table-name messages
```


---


We can get the list of all tables in the selected region (you can change 🔧 region using `aws configure` ):

```sh
aws dynamodb list-tables
```


---

## 2. Create IAM role to access DynamoDB

This is a trust policy. It has a statement field:

```js
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```


---

Let's create an IAM role with this trust policy so our lambda can access DynamoDB. First, create a role with a trust policy from a file using this command `aws iam` which points to a file (you can get it form [the GitHub repository for the AWS Intermediate course](https://github.com/azat-co/aws-intermediate/tree/master/code/serverless)):

```sh
aws iam create-role --role-name LambdaServiceRole --assume-role-policy-document file://lambda-trust-policy.json
```


---


If you are curious, the file `lambda-trust-policy.json` has the lambda service identifier `lambda.amazonaws.com`:

```js
{
    "Role": {
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": "sts:AssumeRole",
                    "Principal": {
                        "Service": [
                            "lambda.amazonaws.com"
                        ]
                    },
                    "Effect": "Allow",
                    "Sid": ""
                }
            ]
        },
        "RoleId": "AROAJLHUFSSSWHS5XKZOQ",
        "CreateDate": "2017-04-26T15:22:41.432Z",
        "RoleName": "LambdaServiceRole",
        "Path": "/",
        "Arn": "arn:aws:iam::161599702702:role/LambdaServiceRole"
    }
}
```


---


Write down the role Arn somewhere. We'll need it later.

Next, add the policies so the lambda function can work with the database:

```sh
aws iam attach-role-policy --role-name LambdaServiceRole --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
```

No output is a good thing in this case. 😄

---



Other optional managed policy which we can use in addition to `AmazonDynamoDBFullAccess` is `AWSLambdaBasicExecutionRole`. It has the logs (CloudWatch) write permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

The commands to attach more managed policies are the same — `aws iam attach-role-policy`.


---


## 3. Create AWS Lambda



---


Now, here's the code for the function. It's in `code/serverless/index.js` in [the GitHub repository for the AWS Intermediate course](https://github.com/azat-co/aws-intermediate/tree/master/code/serverless) . It is very similar to Express request handler. It checks HTTP methods and performs CRUD on DynamoDB table accordingly. Table name comes from query string or from body.

```js
'use strict'

console.log('Loading function')
const doc = require('dynamodb-doc')
const dynamo = new doc.DynamoDB()

// All the request info in event
// "handler" is defined on the function creation
exports.handler = (event, context, callback) => {

    // Callback to finish response
    const done = (err, res) => callback(null, {
        statusCode: err ? '400' : '200',
        body: err ? err.message : JSON.stringify(res),
        headers: {
            'Content-Type': 'application/json',
        }
    })
    // To support mock testing, accept object not just strings
    if (typeof event.body == 'string')
        event.body = JSON.parse(event.body)
    switch (event.httpMethod) {
        // Table name and key are in payload
        case 'DELETE':
            dynamo.deleteItem(event.body, done)
            break
        // No payload, just a query string param    
        case 'GET':
            dynamo.scan({ TableName: event.queryStringParameters.TableName }, done)
            break
        // Table name and key are in payload
        case 'POST':
        //validation
            dynamo.putItem(event.body, done)
            break
        // Table name and key are in payload   
        case 'PUT':
            dynamo.updateItem(event.body, done)
            break
        default:
            done(new Error(`Unsupported method "${event.httpMethod}"`))
    }
}
```



---


So either copy or type the code into a file and archive it with ZIP into `db-api.zip`. That's right. We'll be uploading a zip file to the cloud!


---


## Create an AWS Lambda function from the source code

```
aws lambda create-function --function-name db-api \
  --runtime nodejs6.10 --role arn:aws:iam::161599702702:role/LambdaServiceRole \
  --handler index.handler \
  --zip-file fileb://db-api.zip \
  --memory-size 512 \
  --timeout 10
```

---

Memory size and timeout are optional. By default, they are 128 and 3 correspondingly.


---


```js
{
    "CodeSha256": "bEsDGu7ZUb9td3SA/eYOPCw3GsliT3q+bZsqzcrW7Xg=",
    "FunctionName": "db-api",
    "CodeSize": 778,
    "MemorySize": 512,
    "FunctionArn": "arn:aws:lambda:us-west-1:161599702702:function:db-api",
    "Version": "$LATEST",
    "Role": "arn:aws:iam::161599702702:role/LambdaServiceRole",
    "Timeout": 10,
    "LastModified": "2017-04-26T21:20:11.408+0000",
    "Handler": "index.handler",
    "Runtime": "nodejs6.10",
    "Description": ""
}
```


---

Test function with this data which mocks an HTTP request (`db-api-test.json` file):

```json
{
    "httpMethod": "GET",
    "queryStringParameters": {
        "TableName": "messages"
     }
}
```


---


Run from a CLI (recommended) to execute function in the cloud:

```
aws lambda invoke \
--invocation-type RequestResponse \
--function-name db-api \
--payload file://db-api-test.json \
output.txt
```


---

Or testing can be done from the web console in Lambda dashboard (blue test button once you navigate to function detailed view):

![](http://nodeu.s3.amazonaws.com/images/blog/aws-serverless/serverless-1.png)


---


The results should be 200 (ok status) and output in the `output.txt` file. For example, I do NOT have any record yet so my response is this:

```
{"statusCode":"200","body":"{\"Items\":[],\"Count\":0,\"ScannedCount\":0}","headers":{"Content-Type":"application/json"}}
```


---


The function is working and fetching from the database.  We must test other HTTP methods by modifying the input.  For example, to test creation of an item:

```json
{
  "httpMethod": "POST",
  "queryStringParameters": {
    "TableName": "messages"
  },
  "body": {
    "TableName": "messages",
    "Item":{
       "id":"1",       
       "author": "Neil Armstrong",
       "text": "That is one small step for (a) man, one giant leap for mankind"
    }
  }
}
```


---

## 4. Create API Gateway

We will need to do the following:

1. Create REST API in API Gateway
1. Create a resource (i.e, `/db-api`, e.g.,`/users`, `/accounts`)
1. Define HTTP method(s) without auth
1. Define integration to Lambda (proxy)
1. Create deployment
1. Give permissions for API Gateway resource and method to invoke Lambda


---


The process is not straightforward. Thus, we can use a shell script which will perform all the steps (recommended) or web console.


---

<https://github.com/azat-co/aws-intermediate/tree/master/code/serverless>:

```
sh create-api.sh
```


---



```sh
sh create-api.sh
{
    "id": "sdzbvm11w6",
    "name": "api-for-db-api",
    "description": "Api for db-api",
    "createdDate": 1493242759
}
API ID: sdzbvm11w6
Parent resource ID: sdzbvm11w6
{
    "path": "/db-api",
    "pathPart": "db-api",
    "id": "yjc218",
    "parentId": "xgsraybhu2"
}
Resource ID for path db-api: sdzbvm11w6
{
    "apiKeyRequired": false,
    "httpMethod": "ANY",
    "authorizationType": "NONE"
}
Lambda Arn: arn:aws:lambda:us-west-1:161599702702:function:db-api
{
    "httpMethod": "POST",
    "passthroughBehavior": "WHEN_NO_MATCH",
    "cacheKeyParameters": [],
    "type": "AWS_PROXY",
    "uri": "arn:aws:apigateway:us-west-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-west-1:161599702702:function:db-api/invocations",
    "cacheNamespace": "yjc218"
}
{
    "id": "k6jko6",
    "createdDate": 1493242768
}
APIARN: arn:aws:execute-api:us-west-1:161599702702:sdzbvm11w6
{
    "Statement": "{\"Sid\":\"apigateway-db-api-any-proxy-9C30DEF8-A85B-4EBC-BBB0-8D50E6AB33E2\",\"Resource\":\"arn:aws:lambda:us-west-1:161599702702:function:db-api\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"apigateway.amazonaws.com\"},\"Action\":[\"lambda:InvokeFunction\"],\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:execute-api:us-west-1:161599702702:sdzbvm11w6/*/*/db-api\"}}}"
}
{
    "responseModels": {},
    "statusCode": "200"
}
Resource URL is https://sdzbvm11w6.execute-api.us-west-1.amazonaws.com/prod/db-api/?TableName=messages
Testing...
{"Items":[],"Count":0,"ScannedCount":0}%
```


---

We are all done!


---


## 5. Test

---


```sh
curl "https://sdzbvm11w6.execute-api.us-west-1.amazonaws.com/prod/db-api/?TableName=messages"
```


---


```sh
curl "https://sdzbvm11w6.execute-api.us-west-1.amazonaws.com/prod/db-api/?TableName=messages" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"TableName": "messages",
    "Item": {
      "id": "'$(uuidgen)'",
      "author": "Neil Armstrong",
      "text": "That is one small step for (a) man, one giant leap for mankind"
    }
  }'
```


---


Execute this once to store the env var `API_URL`:

```sh
APINAME=api-for-db-api
REGION=us-west-1
NAME=db-api
APIID=$(aws apigateway get-rest-apis --query "items[?name==\`${APINAME}\`].id" --output text --region ${REGION})
API_URL="https://${APIID}.execute-api.${REGION}.amazonaws.com/prod/db-api/?TableName=messages"
```


---


Then, run CURL for a GET request as many times as you want:

```sh
curl $API_URL
```

And for POST as many times as you want (thanks to `uuidgen`):

```sh
curl ${API_URL} \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"TableName": "messages",
    "Item": {
      "id": "'$(uuidgen)'",
      "author": "Neil Armstrong",
      "text": "That is one small step for (a) man, one giant leap for mankind"
    }
  }'
```


---


The new items can be observed via HTTP interface by making another GET request... or in web console in DynamoDB dashboard as shown below:

![inline](http://nodeu.s3.amazonaws.com/images/blog/aws-serverless/serverless-db.png)



---



![inline](http://nodeu.s3.amazonaws.com/images/blog/aws-serverless/serverless-postman.png)


---


To delete an item with DELETE HTTP request method, the payload must have a `Key`:

```json
{
    "TableName": "messages",
    "Key":{
       "id":"8C968E41-E81B-4384-AA72-077EA85FFD04"
    }

}
```


---

Congratulations! 👏🎆🎉 We've built an event-driven REST API for an entire database not just a single table!


---

Note: For auth, we can set up token-based auth on a resource and method in API Gateway. We can set up response and request rules in the API Gateway as well. Also, everything (API Gateway, Lambda and DynamoDB) can be set up in CloudFormation instead of a CLI or web console ([example of Lambda with CloudFormation](https://github.com/awslabs/lambda-refarch-webapp)).


---

## 6. Clean up


---

Remove API Gateway API with `delete-rest-api`. For example here's my command (for yours replace REST API ID accordingly):

```sh
aws apigateway delete-rest-api --rest-api-id sdzbvm11w6
```


---


Delete the function by its name:

```sh
aws lambda delete-function --function-name db-api
```


---


Finally, delete the database too by its name:

```sh
aws dynamodb delete-table --table-name messages
```


---

## Wrap-up


---

# Some of the frameworks

* Serverless
* Claudia.js

---


For more AWS tutorials, there are other posts in this series on Amazon Web Services:

* [AWS EC2 Web Console Tutorial: Creating WordPress in Minutes](https://node.university/blog/992808/aws-ec2-wordpress)
* [AWS EC2 Web Console Tutorial: Node.js Hello World Server](https://node.university/blog/1001486/aws-ec2-hello-node)
* [AWS EC2 Tutorial: Adding Robustness and Scalability with Elastic Load Balancer](https://node.university/blog/1025425/aws-elb)
* [AWS S3 Tutorial: Easy Static Website Hosting in Under 5 Minutes](https://node.university/blog/1043674/aws-s3)
* [AWS EC2 Autoscaling: Creating an EC2 Autoscaling Group](https://node.university/blog/1061273/aws-autoscaling)
* [AWS Node SDK which Runs EC2](https://node.university/blog/1156474/aws-sdk)
* [AWS CLI Tutorial: Creating a Web Server](https://node.university/blog/1077866/aws-cli)
* [Deploying Node and Mongo Containers on Amazon Web Services Elastic Container Service (AWS ECS)](https://node.university/blog/978472/aws-ecs-containers)



---

Lastly, make sure to checkout some free preview lectures of NodeU courses: 

* [*AWS Intro*](https://node.university/p/aws-intro): Build Solid Foundation of Main AWS Concepts and Services
* [*AWS Intermediate*](https://node.university/p/aws-intermediate): All you need to know to start DevOps with AWS
* [*Node in Production with Docker and AWS at Node University*](https://node.university/p/node-in-production): Learn How to Create and Deploy Container Images for Node.js, MongoDB and Node Stack


---


![inline](http://nodeu.s3.amazonaws.com/images/blog/aws-intermediate.png)

<https://node.university/p/aws-intermediate>


---

## Links

* <http://azat.co>
* <https://webapplog.com> 
* <https://node.university>



