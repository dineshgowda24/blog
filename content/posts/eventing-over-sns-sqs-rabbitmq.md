---
title: "Eventing over SNS-SQS and RabbitMQ"
date: 2022-02-13T10:34:23+05:30
draft: false
categories: [distributed systems]
tags: [rabbitmq, sqs, sns, aws]
---

In this post we will talk about some interesting and powerful use cases of event driven systems that can be solved using RabbitMQ and SNS - SQS combo.

### Goal

Say we have number of microservices running in production and each service encloses a business entity. 

- We want to build a system where any microservice can subscribe to change on any business entity.
- The system in place should be flexible enough that the subscribers can subscribe to specific action on the entity like `CREATE`, `UPDATE` and `DELETE`.
- Any new subscription should be created with minimal code change and no infra change.
- Every message should have some metadata about the producer, type of the entity and the action on the entity.

Eg: In an ecommerce domain, say a notification service which is responsible for sending notification across various platforms, and it wants to subscribe to all the changes on an `Order` such as `CREATE`, `UPDATE`. 
Contrarily, another service may only want to subscribe to a change when a `User` is `DELETED` from the system.


### Prerequisites

- Working knowledge RMQ, SNS, SQS
- Docker 
- Go
- AWS account


### SNS

Amazon Simple Notification Service (Amazon SNS) is a fully managed messaging service for both service-to-service and service-to-person communication. 
The pub/sub functionality provides topics for high-throughput, push-based, many-to-many messaging between microservices, and event-driven serverless applications. 

#### Publishing messages to SNS

Below is an example of `user service` which is publishing a message to `SNS` when an `user` is `updated`.

We will be publishing message to an SNS topic with message attributes such as
- Name of the service which published the message : `userservice`
- Type of the entity : `user`
- Action : `updated`

```go 
package main

import (
	"encoding/json"
	"log"

	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/sns"
)

// User simulates the User in a system
type User struct {
	Name string `json:"name"`
}

func main() {
	topicArn := "arn:aws:sns:ap-south-1:*:events"

	// Initialize a session that the SDK will use to load
	// credentials from the shared credentials file. (~/.aws/credentials).
	sess := session.Must(session.NewSessionWithOptions(session.Options{
		SharedConfigState: session.SharedConfigEnable,
		Profile:           "dinesh", // my AWS profile
	}))

	svc := sns.New(sess)
	raw, err := json.Marshal(&User{
		Name: "Dinesh",
	})
	if err != nil {
		log.Fatal(err)
	}

	jsonStr := string(raw)
	result, err := svc.Publish(&sns.PublishInput{
		Message:           &jsonStr,
		TopicArn:          &topicArn,
		// hardcoded is bad practice, this is just for the purpose of demonstration
		// we are likely to store the service name in ENV, get the struct name using reflection and send action based on the action performed
		MessageAttributes: getMessageAttributes("userservice", "user", "update"),
	})
	if err != nil {
		log.Fatal(err)
	}

	log.Println(*result.MessageId)
}

// getMessageAttributes returns message attributes which encloses sender of the event, entity name, action
func getMessageAttributes(sender, entityName, action string) map[string]*sns.MessageAttributeValue {
	attributes := make(map[string]*sns.MessageAttributeValue)
	stringType := "String"
	attributes["sender"] = &sns.MessageAttributeValue{
		DataType:    &stringType,
		StringValue: &sender,
	}

	attributes["entity"] = &sns.MessageAttributeValue{
		DataType:    &stringType,
		StringValue: &entityName,
	}

	attributes["action"] = &sns.MessageAttributeValue{
		DataType:    &stringType,
		StringValue: &action,
	}
	return attributes
}
```

#### SNS Subscription & filter policy

SNS subscription offers subscription mechanism by which messages can be sent to `SQS`.
Filter policy are cherry on top which allow you to filter specific message based on message attributes.
These filter policies are just regular `JSON` text. A consumer who wishes to consume events from `userservice` when a `user` is `updated` or `created` should add the below filter policy

```json
{
  "entity": [
    "user"
  ],
  "sender": [
    "userservice"
  ],
  "action": [
    "update",
    "create"
  ]
}
```

### SQS

Amazon Simple Queue Service (SQS) is a fully managed message queuing service.
SQS offers two types of message queues. 
- Standard queues offer maximum throughput, best-effort ordering, and at-least-once delivery. 
- SQS FIFO queues are designed to guarantee that messages are processed exactly once, in the exact order that they are sent.


#### SQS Consumer

Once the SQS with subscription and filter policy is set up. We would need the queue url to consume. Below is the example of consumer messages from `SQS`.

```go
package main

import (
	"context"
	"log"
	"time"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/sqs"
)

func GetQueueURL(sess *session.Session, queue *string) (*sqs.GetQueueUrlOutput, error) {
	svc := sqs.New(sess)
	urlResult, err := svc.GetQueueUrl(&sqs.GetQueueUrlInput{
		QueueName: queue,
	})
	if err != nil {
		return nil, err
	}
	return urlResult, nil
}

func GetMessages(sess *session.Session, queueURL *string, timeout , waitSeconds *int64) (*sqs.ReceiveMessageOutput, error) {
	// Create an SQS service client
	svc := sqs.New(sess)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*time.Duration(*timeout+5))
	defer cancel()

	msgResult, err := svc.ReceiveMessageWithContext(ctx,
		&sqs.ReceiveMessageInput{
			AttributeNames: []*string{
				aws.String(sqs.MessageSystemAttributeNameSentTimestamp),
			},
			MessageAttributeNames: []*string{
				aws.String(sqs.QueueAttributeNameAll),
			},
			QueueUrl:            queueURL,
			MaxNumberOfMessages: aws.Int64(1),
			VisibilityTimeout:   timeout,
			WaitTimeSeconds: waitSeconds,
		})
	if err != nil {
		return nil, err
	}
	return msgResult, nil
}

func main() {
	// Create a session that gets credential values from ~/.aws/credentials
	// and the default region from ~/.aws/config
	sess := session.Must(session.NewSessionWithOptions(session.Options{
		SharedConfigState: session.SharedConfigEnable,
		Profile:           "dinesh",
	}))

	// Get URL of queue
	queueName := "events"
	urlResult, err := GetQueueURL(sess, &queueName)
	if err != nil {
		log.Fatal(err)
	}

	queueURL := urlResult.QueueUrl
	waitSeconds, timeout := int64(20), int64(60)
	for {
		log.Println("polling for messages")
		msgResult, err := GetMessages(sess, queueURL, &timeout, &waitSeconds)
		if err != nil {
			if err == context.DeadlineExceeded {
				continue
			}
			log.Fatal(err)
		}
		for _, msg := range msgResult.Messages {
			log.Println("Message Body : ", *msg.Body)
		}
	}
}
```

### RMQ

RMQ open source message broker. It supports wide variety of protocols such as AMQP 0-9-1, STOMP, MQTT, AMQP 1.0. It also comes with support of web based
monitoring build in. 

The most powerful of them all is the dynamic message routing capability using `topic` exchange.

#### Starting RMQ On Docker

```shell
docker run -d --hostname rmq --name rmq -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=root -e RABBITMQ_DEFAULT_PASS=root rabbitmq:3-management
```

#### Exchanges & Publishing Messages

Replicating the same thing we did for SNS using RMQ. 
Below is an example which creates a `topic` exchange called `events`. It also publishes a message to exchange with a routing key in the format `<service>.<entityname>.<action>`

```go
package main

import (
	"encoding/json"
	"log"

	"github.com/streadway/amqp"
)

type User struct {
	Name string `json:"name"`
}

func main() {
	conn, err := amqp.Dial("amqp://root:root@localhost:5672/")
	if err != nil {
		log.Fatalf("%s: %s", "Failed to connect to RabbitMQ", err)
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("%s: %s", "Failed to open a channel", err)
	}
	defer ch.Close()

	err = ch.ExchangeDeclare(
		"events",     // name
		"topic",      // type
		true,         // durable
		false,        // auto-deleted
		false,        // internal
		false,        // no-wait
		nil,          // arguments
	)
	if err != nil {
		log.Fatalf("%s: %s", "Failed to declare an exchange", err)
	}

	user := User{"Dinesh"}
	raw, err := json.Marshal(&user)
	if err != nil {
		log.Fatalf("%s: %s", "Failed to marshal message", err)
	}
	
	err = ch.Publish(
		"events",                   // exchange
		"userservice.user.updated", // routing key <service name>.<entity name>.<action>
		false,                      // mandatory
		false,                      // immediate
		amqp.Publishing{
			ContentType: "text/json",
			Body:        raw,
		})
	if err != nil {
		log.Fatalf("%s: %s", "Failed to publish a message", err)
	}

	log.Println("Message published successfully")
}
```

#### Consuming Messages

The powerful feature of `topic` exchange is while binding the queue we can specify wild card characters delimited by `.`.
A consumer wanting to subscribe to event from `userservice` when a `user` is created, updated or deleted will use the routing key `userservice.user.*`

```go
package main

import (
	"log"

	"github.com/streadway/amqp"
)

func main() {
	conn, err := amqp.Dial("amqp://root:root@localhost:5672/")
	if err != nil {
		log.Fatalf("%s: %s", "Failed to connect to RabbitMQ", err)
	}

	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("%s: %s", "Failed to connect to RabbitMQ", err)
	}

	defer ch.Close()

	err = ch.ExchangeDeclare(
		"events",                // name
		amqp.ExchangeTopic,      // type
		true,                    // durable
		false,                   // auto-deleted
		false,                   // internal
		false,                   // no-wait
		nil,                     // arguments
	)
	if err != nil {
		log.Fatalf("%s: %s", "Failed to connect to RabbitMQ", err)
	}

	q, err := ch.QueueDeclare(
		"notificationservice-userevents",    // name
		true,                                // durable, survives broker restart
		false,                               // delete when unused
		false,                               // exclusive
		false,                               // no-wait
		nil,                                 // arguments
	)
	if err != nil {
		log.Fatalf("%s: %s", "Failed to declare queue", err)
	}

	routingKey := "userservice.user.*"
	log.Printf("Binding queue %s to exchange %s with routing key %s", q.Name, "events", routingKey)
	err = ch.QueueBind(
		q.Name,       // queue name
		routingKey,   // routing key
		"events",     // exchange
		false,
		nil)
	if err != nil {
		log.Fatalf("%s: %s", "Failed to bind queue to exchange", err)
	}

	msgs, err := ch.Consume(
		q.Name, 	// queue
		"",     	// consumer
		true,   	// auto ack
		false,  	// exclusive
		false,  	// no local
		false,  	// no wait
		nil,        // args
	)
	if err != nil {
		log.Fatalf("%s: %s", "Failed to connect to consume", err)
	}

	forever := make(chan bool)

	go func() {
		for d := range msgs {
			log.Printf(" Message Body : %s", d.Body)
		}
	}()

	log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
	<-forever
}
```

##### Note

_Some things in the code is deliberately left out in the interest of time and complexity._