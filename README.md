# Choria Lifecycle Events

This package create and view Choria Lifecycle Events

These lifecycle events are published to the `choria.lifecycle.event.<type>.<component>` topic structure of the middleware and contains small JSON documents that informs listeners about significant life cycle events of Choria components.

[![GoDoc](https://godoc.org/github.com/choria-io/go-lifecycle?status.svg)](https://godoc.org/github.com/choria-io/go-lifecycle) [![CircleCI](https://circleci.com/gh/choria-io/go-lifecycle/tree/master.svg?style=svg)](https://circleci.com/gh/choria-io/go-lifecycle/tree/master)

## Status

This project is versioned using SemVer and have reached version 1.0.0, it's in use by several Choria projects and will follow SemVer rules in future.

## Supported Events

|Event|Description|
|-----|-----------|
|Startup|Event to emit when components start, requires `Identity()`, `Component()` and `Version()` options|
|Shutdown|Event to emit when components shut down, requires `Identity()` and `Component()` options|
|Provisioned|Event to emit after provisioning of a component, requires `Identity()` and `Component()` options|
|Alive|Event to emit at regular intervals indicating it's still functional, requires `Identity()`, `Component()` and `Version()` options|

#### Sample Events
### Schemas

Event Schemas are stored in the [Choria Schemas repository](https://github.com/choria-io/schemas/tree/master/choria/lifecycle).

#### Startup

```json
{
    "protocol":"io.choria.lifecycle.v1.startup",
    "id":"01e72410-d734-4611-9485-8c6a2dd2579b",
    "identity":"c1.example.net",
    "version":"0.6.0",
    "timestamp":1535369537,
    "component":"server"
}
```

#### Shutdown

```json
{
    "protocol":"io.choria.lifecycle.v1.shutdown",
    "id":"01e72410-d734-4611-9485-8c6a2dd2579b",
    "identity":"c1.example.net",
    "component":"server",
    "timestamp":1535369536
}
```

#### Provisioned

```json
{
    "protocol":"io.choria.lifecycle.v1.provisioned",
    "id":"01e72410-d734-4611-9485-8c6a2dd2579b",
    "identity":"c1.example.net",
    "component":"server",
    "timestamp":1535369536
}
```

#### Alive

```json
{
    "protocol":"io.choria.lifecycle.v1.alive",
    "id":"01e72410-d734-4611-9485-8c6a2dd2579b",
    "identity":"c1.example.net",
    "version":"0.6.0",
    "timestamp":1535369537,
    "component":"server"
}
```

## Viewing events

In a shell configured as a Choria Client run `choria tool event` to view events in real time. You can also install the CLI found on our releases page and do `lifecycle view`.

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

## Tallying component versions

In large dynamic fleets it's hard to keep track of counts and versions of nodes. A tool is included that can observe a running network and gather versions of a specific component.  The results are exposed as Prometheus metrics.

```
lifecycle tally --component server --port 8080 --prefix lifecycle_tally
```

For this to work it uses the normal Choria client configuration to connect to the right middleware using TLS and listen there, you'll .

This will listen on port 8080 for `/metrics`, it will observe events from the `server` component and expose metrics as below:

|Metric|Description|
|------|-----------|
|lifecycle_tally_good_events|Events processed successfully|
|lifecycle_tally_process_errors|The number of events received that failed to process|
|lifecycle_tally_event_types|The number of events received by type|
|lifecycle_tally_versions|Gauge indicating the number of running components by version|
|lifecycle_tally_maintenance_time|Time spent doing regular maintenance on the stored data|
|lifecycle_tally_processing_time|The time taken to process events|

Additionally this tool can also watch Choria Autonomous Agent events, today it supports transition events only:

|Metric|Description|
|------|-----------|
|lifecycle_tally_machine_transition|Information about transition events handled by Choria Autonomous Agents|

Here the prefix - `lifecycle_tally` - is what would be the default if you didn't specify `--prefix`.