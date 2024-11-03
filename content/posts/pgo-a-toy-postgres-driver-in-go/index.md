---
title: "Pgo: Writing a Postgres Driver from Scratch in Go"
date: 2024-09-18T23:19:00+05:30
draft: true
---

## Preface

A couple of weeks back, I was thinking about benchmarking libpq and pgx driver to understand the performance bottlenecks in both drivers as in our production set up we have been using libpq and I read about Phil's post about pgx being better for production use.

One thing that started bugging me was why do we have two drivers in the first place, I felt that low level communications are trivial and there should have been only drivers. Another thing that also confused me was redis client in go does not have a seperate driver, then what's special about Postgres, which is when I discovered Postgres Wire Protocol.

I started reading about the protocol itself and so many things started to resonate with me and things started becoming clearer. I loved the protocol and eventually decided to write a toy driver to understand lower level workings of the protocol itself.

As decided to write a toy driver, I did not have any idea about how to start, so I started reading about the protocol itself and started writing a simple client to connect to the server and send a simple query. Since I have no prior experience in writing a driver, I did go through the source code of libpq and pgx to understand how they are implemented and how each component within the driver is implemented.

Over the course of time, I did add a flavour of my own to the driver, since performance was not my primary goal, I focused on making it easier to read and follow along.

## Tld'r

If are someone who already has a understanding of how the protocol works and want to see the driver, you can directly jump to the [*Github repository*](https://github.com/dineshgowda24/pgo).

## Overview of the Postgres Wire Protocol

The protocol itself is very simple and easy to understand, it is a TCP based protocol. Every data that is sent or received is a type of message. There are two types of messages. A frontend message and a backend message. A frontend message is sent by the client(our Go driver) to the server(Postgres) and a backend message is sent by the server(Postgres) to the client(our Go driver).

### Message Format

Every message starts with a single byte which indicates message type, followed by a 4 byte message length(including the length of self) and the actual message body. The message body is different for each message type and can be of any length.

<img src="images/msg_format.svg" width= "40%" style="border:none;" alt="b-kv"/>

Except for the [StartupMessage](https://www.postgresql.org/docs/13/protocol-message-formats.html#PROTOCOL-FRONTEND-MESSAGE-TYPE), 
all other messages follow the same format.

### StartupMessage

**Purpose**

The StartupMessage is the first message sent by the client to initiate a connection with the PostgreSQL server. It includes the protocol version, followed by a set of parameters like the username and database name. In this driver, we use protocol version 3.0 for the connection.

**Sending the Startup Message**

The sendStartup function below constructs the StartupMessage by creating a map of parameters, such as the username and database name, and sends it using the send method.

```go
func (c *Conn) sendStartup() error {
    params := map[string]string{
        "user":     c.conf.username,
        "database": c.conf.dbname,
    }
    return c.send(pg.NewStartupMessage(params))
}
```

**Structuring the StartupMessage**

The **StartupMessage** struct defines the **ProtocolVersion** and **Parameters** fields. Note that the protocol version `3.0` is encoded as `0x30000`, where the first `16` bits represent the major version `(3)` and the last 16 bits represent the minor version `(0)`.

```go
var _ Message = (*StartupMessage)(nil)

// ProtocolVersion represents 3.0, which is encoded as 0x30000
// The 1st 16 bits represent the major version (3) and the last 16 bits represent the minor version (0)
const ProtocolVersion = 0x30000

type StartupMessage struct {
	ProtocolVersion int32
	Parameters      map[string]string
	MsgType         byte
}
```

Below is byte level representation of the StartupMessage.

<img src="images/startup_msg.svg" width= "80%" style="border:none;" alt="b-kv"/>


**Legacy Design Choice**

Interestingly, the StartupMessage does not contain a message type. This is mostly a legacy feature; the server implicitly knows that the first message sent by the client is the StartupMessage.

**Handling Authentication**

```go
case pg.MessageTypeAuthenticationMessage:
    if err := c.a.Authenticate(c, body); err != nil {
        return err
    }
```

**Query Execution – Basic vs. Prepared Queries**

```go
func (c *Conn) basicQuery(query string) (*Rows, error) {
    if err := c.send(pg.NewQuery(query)); err != nil {
        return nil, err
    }
    mt, msg, err := c.recvMsg()
    ...
    return &Rows{conn: c, desc: msg.(*pg.RowDescription)}, nil
}
```

```go
func (c *Conn) preparedQuery(query string, args ...interface{}) (*Rows, error) {
    values, err := c.typeConv.ToBytes(args...)
    if err != nil {
        return nil, fmt.Errorf("failed to convert args to bytes: %w", err)
    }
    msgs := []pg.Message{
        pg.NewParse("", query, nil),
        pg.NewBind("", "", inferFormatCode(args...), values, nil),
        pg.NewDescribe(pg.TargetTypePortal, ""),
        pg.NewExecute("", 0),
        pg.NewSync(),
    }
    ...
}
```

**Connection Readiness**

```go
func (c *Conn) ready() error {
    mt, msg, err := c.recvMsg()
    if err != nil {
        return err
    }
    if mt == pg.MessageTypeReadyForQuery {
        c.txn = TxnStatus(msg.(*pg.ReadyForQuery).TxStatus)
        return nil
    }
    c.txn = Failed
    return msgTypeErr("ready check failed", mt)
}
```

**Driver High Level Overview**

<img src="images/hld.svg" width= "80%" style="border:none;" alt="hld"/>