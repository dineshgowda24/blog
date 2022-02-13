---
title: "RabbitMQ Over SNS SQS"
date: 2022-02-13T10:34:23+05:30
draft: true
---

In this post we will talk about some interesting and powerful use cases that can be solved using RMQ in an event driven system and limitations with SNS and SQS combo for the same use case.

### Use Case

Let's say we have number of microservices running in production and each service encloses a business entity. 
- We want to build a system where any microservice can subscribe to change on any business entity with minimal code change and practically no infra change. 
- The system in place should be flexible enough that the subscribers can subscribe to specific CRUD operation. 
- The consumer should be dumb enough and should ask for what is needed and the system inplace should be smart enough to deliver the ask.

Eg: In an ecommerce domain we, say a notification service wants to subscribe to all the changes on an `Order` such as CREATE, UPDATE. Contrarily, a service may only want to subscribe to a change when a `User` is DELETED from the system.

### SNS

Amazon Simple Notification Service (Amazon SNS) is a fully managed messaging service for both service-to-service and service-to-person communication.

The pub/sub functionality provides topics for high-throughput, push-based, many-to-many messaging between microservices, and event-driven serverless applications. 
Using Amazon SNS topics, publishers can fanout messages to a large number of subscribers. The subscribers include Amazon SQS queues, AWS Lambda functions, HTTPS endpoints. The service-to-person functionality enables you to send messages to users at scale via SMS, mobile push, and email.

### SQS

Amazon Simple Queue Service (SQS) is a fully managed message queuing service that enables you to decouple and scale microservices, distributed systems, and serverless applications.
SQS offers two types of message queues. Standard queues offer maximum throughput, best-effort ordering, and at-least-once delivery. SQS FIFO queues are designed to guarantee that messages are processed exactly once, in the exact order that they are sent.


### Publishing messages to SNS

```go
package main

import (
    "encoding/json"
    "github.com/aws/aws-sdk-go-v2/aws/external"
    "github.com/aws/aws-sdk-go-v2/service/sns"
    "github.com/aws/aws-sdk-go/aws"
)

type User struct {
    Name string `json:"name"`
} 

func main() {
    cfg, _ := external.LoadDefaultAWSConfig()
    snsClient := sns.New(cfg)
    
    // Some business logic which updates the User
    person := User{
        Name:"Dinesh Gowda",
    }
    jsonStr, _ := json.Marshal(person)

    req := snsClient.PublishRequest(&sns.PublishInput{
        TopicArn: aws.String("arn:aws:sns:us-east-1:*****:ok"),
        Message: aws.String(string(jsonStr)),
        MessageStructure: aws.String("json"),
        MessageAttributes: map[string]sns.MessageAttributeValue{
            "name": {
                DataType: aws.String("String"),
                StringValue: aws.String("User"),
            },
			"action": {
				DataType: aws.String("String"),
				StringValue: aws.String("Created"),
			},
        },
    })

    req.Send()
}
```

### RMQ

RMQ open source message broker. It supports wide variety of protocols such as AMQP 0-9-1, STOMP, MQTT, AMQP 1.0. It also comes with support of web based
monitoring build in. The most powerful of them all is the dynamic message routing capability.
