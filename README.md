# Choria Lifecycle Events

This package create and view Choria Lifecycle Events

These lifecycle events are published to the `choria.lifecycle.event.<type>.<component>` topic structure of the middleware and contains small JSON documents that informs listeners about significant life cycle events of Choria components.

[![GoDoc](https://godoc.org/github.com/choria-io/go-lifecycle?status.svg)](https://godoc.org/github.com/choria-io/go-lifecycle) [![CircleCI](https://circleci.com/gh/choria-io/go-lifecycle/tree/master.svg?style=svg)](https://circleci.com/gh/choria-io/go-lifecycle/tree/master)

## Supported Events

|Event|Description|
|-----|-----------|
|Startup|Event to emit when components start, requires `Identity()`, `Component()` and `Version()` options|
|Shutdown|Event to emit when components shut down, requires `Identity()` and `Component()` options|
|Provisioned|Event to emit after provisioning of a component, requires `Identity()` and `Component()` options|

#### Sample Events
### Schemas

Event Schemas are stored in the [Choria Schemas repository](https://github.com/choria-io/schemas/tree/master/choria/lifecycle).

#### Startup

```json
{
    "protocol":"choria:lifecycle:startup:1",
    "identity":"c1.example.net",
    "version":"0.6.0",
    "timestamp":1535369537,
    "component":"server"
}
```

#### Shutdown

```json
{
    "protocol":"choria:lifecycle:shutdown:1",
    "identity":"c1.example.net",
    "component":"server",
    "timestamp":1535369536
}
```

#### Provisioned

```json
{
    "protocol":"choria:lifecycle:provisioned:1",
    "identity":"c1.example.net",
    "component":"server",
    "timestamp":1535369536
}
```

## Viewing events

In a shell configured as a Choria Client run `choria tool event` to view events in real time.

These events do not traverse Federation borders, so you have to view them in the network you care to observe.  You can though configure a Choria Adapter to receive them and adapt them onto a NATS Stream from where you can replicate them to other data centers.

## Emitting an event

```go
event, err := lifecycle.New(lifecycle.Startup, lifecycle.Identity("my.identity"), lifecycle.Component("my_app"), lifecycle.Version("0.0.1"))
panicIfErr(err)

// conn is a Choria connector
err = lifecycle.PublishEvent(event, conn)
```

If you are emitting `lifecycle.Shutdown` events right before exiting be sure to call `conn.Close()` so the buffers are flushed prior to shutdown.

## Receiving events

These events are used to orchestrate associated tools like the [Provisioning Server](https://github.com/choria-io/provisioning-agent) that listens for these events and immediately add a new node to the provisioning queue.

To receive `startup` events for the `server`:

```go
events := make(chan *choria.ConnectorMessage, 1000)

// conn is a choria framework connector
// fw is the choria framework
err = conn.QueueSubscribe(ctx, fw.NewRequestID(), "choria.lifecycle.event.startup.server", "", events)
panicIfError(err)

for {
    select {
    case e := <-events:
        event, err := lifecycle.NewFromJSON(e.Data)
        if err != nil {
            continue
        }

        fmt.Printf("Received a startup from %s", event.Identity())
    case <-ctx.Done():
        return
    }
}
```