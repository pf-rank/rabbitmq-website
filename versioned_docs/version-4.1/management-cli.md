---
title: rabbitmqadmin v2, a Command Line Tool for the HTTP API
displayed_sidebar: docsSidebar
---
<!--
Copyright (c) 2005-2025 Broadcom. All Rights Reserved. The term "Broadcom" refers to Broadcom Inc. and/or its subsidiaries.

All rights reserved. This program and the accompanying materials
are made available under the terms of the under the Apache License,
Version 2.0 (the "License”); you may not use this file except in compliance
with the License. You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# rabbitmqadmin v2, a Command Line Tool for the HTTP API

[`rabbitmqadmin` v2](https://github.com/rabbitmq/rabbitmqadmin-ng) is a command line tool that uses the HTTP API.

It supports many of the operations available in the management UI:

 * Listing objects (virtual hosts, users, queues, streams, permissions, policies, and so on)
 * Creating objects
 * Deleting objects
 * Access to cluster and node metrics
 * Run [health checks](./monitoring#health-checks)
 * Listing [feature flag](./feature-flags) state
 * Listing [deprecated features](./deprecated-features) in use across the cluster
 * [Definition](./definitions) export, transformations, and import
 * Operations on [shovels](./shovel)
 * Operations on [federation upstreams](./federation) and links
 * Closing connections
 * Rebalancing of queue leaders across cluster nodes

Note that `rabbitmqadmin` is not a replacement for [rabbitmqctl](./man/rabbitmqctl.8) or
[rabbitmq-plugins](./man/rabbitmq-plugins.8) as the HTTP API intentionally doesn't expose certain operations.


## `rabbitmqadmin` v2

[This generation of `rabbitmqadmin`](https://github.com/rabbitmq/rabbitmqadmin-ng) is a standalone project that
has its own development cycle that's independent from that of RabbitMQ.

It is distributed as a native binary.

### Downloads

Binaries for x86-64 Linux, aarch64 Linux, aarch64 macOS and x86-64 Windows
are distributed via [GitHub releases](https://github.com/rabbitmq/rabbitmqadmin-ng/releases).


## Usage

### Exploring Available Command Groups and Sub-commands

To explore what command groups are available, use

```shell
rabbitmqadmin help
```

which will output a list of command groups:

```
Usage: rabbitmqadmin [OPTIONS] <COMMAND>

Commands:
  bindings             Operations on bindings
  channels             Operations on channels
  close                Closes connections
  connections          Operations on connections
  declare              Creates or declares objects
  definitions          Operations on definitions (everything except for messages: virtual hosts, queues, streams, exchanges, bindings, users, etc)
  delete               Deletes objects
  deprecated_features  Operations on deprecated features
  exchanges            Operations on exchanges
  export               See 'definitions export'
  feature_flags        Operations on feature flags
  federation           Operations on federation upstreams and links
  get                  Fetches message(s) from a queue or stream via polling. Only suitable for development and test environments.
  global_parameters    Operations on global runtime parameters
  health_check         Runs health checks
  import               See 'definitions import'
  list                 Lists objects
  nodes                Node operations
  operator_policies    Operations on operator policies
  parameters           Operations on runtime parameters
  passwords            Operations on passwords
  policies             Operations on policies
  publish              Publishes (inefficiently) message(s) to a queue or a stream. Only suitable for development and test environments.
  purge                Purges queues
  queues               Operations on queues
  rebalance            Rebalancing of leader replicas
  show                 Overview, memory footprint breakdown, and more
  shovels              Operations on shovels
  streams              Operations on streams
  tanzu                Tanzu RabbitMQ-specific commands
  users                Operations on users
  vhosts               Virtual host operations
  help                 Print this message or the help of the given subcommand(s)
```

To explore commands in a specific group, use

```shell
rabbitmqadmin {group name} help
```

### Exploring the CLI with `help`, `--help`

To learn about what command groups and specific commands are available, run

``` shell
rabbitmqadmin help
```

This flag can be appended to a command or subcommand to get command-specific documentation:

```shell
rabbitmqadmin queues declare --help
# => creates or declares things
# =>
# => Usage: rabbitmqadmin declare [object]
# => ...
```

Alternatively, the `help` subcommand can be given a command name. It's the equivalent
of tagging on `--help` at the end of command name:

```shell
rabbitmqadmin queues help declare
# => creates or declares things
# =>
# => Usage: rabbitmqadmin declare [object]
# => ...
```

More specific examples are covered in the Examples section below.


### Interactive vs. Use in Scripts

Like the original version, `rabbitmqadmin` v2 is first and foremost built for interactive use
by humans. Many commands will output formatted tables, for example:

```shell
rabbitmqadmin show overview
```

will output a table that looks like this:

```
┌───────────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Overview                                                                                                                                                            │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ key                                                               │ value                                                                                           │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Product name                                                      │ RabbitMQ                                                                                        │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Product version                                                   │ 4.1.4                                                                                           │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ RabbitMQ version                                                  │ 4.1.4                                                                                           │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Erlang version                                                    │ 27.3.4                                                                                          │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Erlang details                                                    │ Erlang/OTP 27 [erts-15.2.7] [source] [64-bit] [smp:10:10] [ds:10:10:10] [async-threads:1] [jit] │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Connections (total)                                               │ 0                                                                                               │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ AMQP 0-9-1 channels (total)                                       │ 0                                                                                               │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Queues and streams (total)                                        │ 1                                                                                               │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Consumers (total)                                                 │ 0                                                                                               │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Messages (total)                                                  │ 0                                                                                               │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Messages ready for delivery (total)                               │ 0                                                                                               │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Messages delivered but unacknowledged by consumers (total)        │ 0                                                                                               │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Publishing (ingress) rate (global)                                │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Publishing confirm rate (global)                                  │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Consumer delivery (egress) rate (global)                          │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Consumer delivery in automatic acknowledgement mode rate (global) │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Consumer acknowledgement rate (global)                            │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Unroutable messages: returned-to-publisher rate (global)          │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Unroutable messages: dropped rate (global)                        │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Cluster tags                                                      │ "core": "8"                                                                                     │
│                                                                   │ "env": "development"                                                                            │
│                                                                   │ "machine": "X8Y33L97DQ"                                                                         │
│                                                                   │                                                                                                 │
├───────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Node tags                                                         │ "environment": "production"                                                                     │
│                                                                   │ "purpose": "iot_ingress"                                                                        │
│                                                                   │ "region": "ca-central-1"                                                                        │
│                                                                   │ "series": "4.1.x"                                                                               │
│                                                                   │                                                                                                 │
└───────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

As it is easy to observe, parsing such output in a script will be challenging.

For this reason, `rabbitmqadmin` v2 can render results in a way that would be much more friendly
for scripting if the `--non-interactive` flag is passed. It is a global flag so it must be
passed before the command and subcommand name:

```shell
rabbitmqadmin --non-interactive show overview
```

The output of the above command will not include any table borders and will is much easier to parse
as a result:

```
key
Product name                                                      RabbitMQ
Product version                                                   4.1.4
RabbitMQ version                                                  4.1.4
Erlang version                                                    27.3.4
Erlang details                                                    Erlang/OTP 27 [erts-15.2.7] [source] [64-bit] [smp:10:10] [ds:10:10:10] [async-threads:1] [jit]
```

### Retrieving Basic Node Information

``` shell
rabbitmqadmin show overview
```

will display essential node information in tabular form.

### Retrieving Connection, Queue/Stream, Channel Churn Information

Helps assess connection, queue/stream, channel [churn metrics](./connections#high-connection-churn) in the cluster.

``` shell
rabbitmqadmin show churn
```

### Listing cluster nodes

``` shell
rabbitmqadmin list nodes
```

### Listing virtual hosts

``` shell
rabbitmqadmin list vhosts
```

### Listing users

``` shell
rabbitmqadmin list users
```

### Create a User

```shell
rabbitmqadmin users declare --name "app-user" --password "secure-password" --tags "monitoring,policymaker"
```

```shell
# Create user with password hash (recommended for security)
rabbitmqadmin users declare --name "app-user" --password-hash "$2b$12$abcdefghijklmnopqrstuvwxyz" --tags "administrator"
```

### Delete a User

```shell
rabbitmqadmin users delete --name "app-user"
```

```shell
# Idempotent deletion (no error if user doesn't exist)
rabbitmqadmin users delete --name "app-user" --idempotently
```

### List User Permissions

```shell
# List permissions for all users
rabbitmqadmin users permissions
```

### List User Connections

```shell
# List connections for a specific user
rabbitmqadmin users connections --name "app-user"
```

### Listing queues

``` shell
rabbitmqadmin list queues
```

``` shell
rabbitmqadmin --vhost "monitoring" list queues
```

### Listing exchanges

``` shell
rabbitmqadmin list exchanges
```

``` shell
rabbitmqadmin --vhost "events" list exchanges
```

### Listing bindings

``` shell
rabbitmqadmin list bindings
```

``` shell
rabbitmqadmin --vhost "events" list bindings
```

### Bind a queue to an exchange

```shell
rabbitmqadmin --vhost "events" bindings declare --source "events.topic" --destination-type "queue" --destination "target.queue" --routing-key "events.order.created"
```

### Bind a source exchange to a destination exchange

```shell
rabbitmqadmin --vhost "events" bindings declare --source "events.topic" --destination-type "exchange" --destination "events.fanout" --routing-key "events.*"
```

### Delete a Binding

```shell
rabbitmqadmin --vhost "events" bindings delete --source "events.topic" --destination-type "queue" --destination "target.queue" --routing-key "events.order.created"
```

### Create a Virtual Host

```shell
rabbitmqadmin vhosts declare --name "vh-789" --default-queue-type "quorum" --description "Used to reproduce issue #789"
```

### Delete a Virtual Host

```shell
rabbitmqadmin vhosts delete --name "vh-789"
```

```shell
# --idempotently means that 404 Not Found responses will not be  considered errors
rabbitmqadmin vhosts delete --name "vh-789" --idempotently
```


### Declare a Queue

```shell
rabbitmqadmin --vhost "events" queues declare --name "target.quorum.queue.name" --type "quorum" --durable true
```

```shell
rabbitmqadmin --vhost "events" queues declare --name "target.stream.name" --type "stream" --durable true
```

### Listing streams

``` shell
rabbitmqadmin list streams
```

``` shell
# List streams in a specific virtual host
rabbitmqadmin --vhost "events" streams list
```

### Declare a Stream

```shell
rabbitmqadmin --vhost "events" streams declare --name "events.stream" --expiration "7D"
```

### Delete a Stream

```shell
rabbitmqadmin --vhost "events" streams delete --name "events.stream"
```

```shell
rabbitmqadmin --vhost "events" queues declare --name "target.classic.queue.name" --type "classic" --durable true --auto-delete false
```

### Purge a queue

```
rabbitmqadmin --vhost "events" queues purge --name "target.queue.name"
```

### Delete a queue

``` shell
rabbitmqadmin --vhost "events" queues delete --name "target.queue.name"
```

``` shell
# --idempotently means that 404 Not Found responses will not be considered errors
rabbitmqadmin --vhost "events" queues delete --name "target.queue.name" --idempotently
```

### Declare an Exchange

```shell
rabbitmqadmin --vhost "events" exchanges declare --name "events.all_types.topic" --type "topic" --durable true
```

```shell
rabbitmqadmin --vhost "events" exchanges declare --name "events.all_type.uncategorized" --type "fanout" --durable true --auto-delete false
```

```shell
rabbitmqadmin --vhost "events" exchanges declare --name "local.random.c60bda92" --type "x-local-random" --durable true
```

### Delete an exchange

``` shell
rabbitmqadmin --vhost "events" exchanges delete --name "target.exchange.name"
```

``` shell
# --idempotently means that 404 Not Found responses will not be  considered errors
rabbitmqadmin --vhost "events" exchanges delete --name "target.exchange.name" --idempotently
```

### Listing connections

``` shell
rabbitmqadmin list connections
```

``` shell
# List connections for a specific virtual host
rabbitmqadmin --vhost "events" connections list
```

### Listing connections for a specific user

``` shell
rabbitmqadmin connections list_of_user --name "app-user"
```

### Close a connection

``` shell
rabbitmqadmin connections close --name "127.0.0.1:5672 -> 127.0.0.1:59876"
```

### Close all connections for a user

``` shell
rabbitmqadmin connections close_of_user --name "app-user"
```

### Listing channels

``` shell
rabbitmqadmin list channels
```

``` shell
# List channels in a specific virtual host
rabbitmqadmin --vhost "events" channels list
```

### Inspecting Node Memory Breakdown

There are two commands for reasoning about target [node's memory footprint](./memory-use):

```shell
# displays a breakdown in bytes
rabbitmqadmin show memory_breakdown_in_bytes --node 'rabbit@hostname'
```

```shell
# displays a breakdown in percent
rabbitmqadmin show memory_breakdown_in_percent --node 'rabbit@hostname'
```

Example output of `show memory_breakdown_in_percent`:

```
┌────────────────────────────────────────┬────────────┐
│ key                                    │ percentage │
├────────────────────────────────────────┼────────────┤
│ total                                  │ 100%       │
├────────────────────────────────────────┼────────────┤
│ Binary heap                            │ 45.10%     │
├────────────────────────────────────────┼────────────┤
│ Allocated but unused                   │ 23.45%     │
├────────────────────────────────────────┼────────────┤
│ Quorum queue ETS tables                │ 23.05%     │
├────────────────────────────────────────┼────────────┤
│ Other processes                        │ 5.32%      │
├────────────────────────────────────────┼────────────┤
│ Other (used by the runtime)            │ 4.98%      │
├────────────────────────────────────────┼────────────┤
│ Code                                   │ 4.54%      │
├────────────────────────────────────────┼────────────┤
│ Client connections: others processes   │ 3.64%      │
├────────────────────────────────────────┼────────────┤
│ Management stats database              │ 3.48%      │
├────────────────────────────────────────┼────────────┤
│ Client connections: reader processes   │ 3.22%      │
├────────────────────────────────────────┼────────────┤
│ Plugins and their data                 │ 3.12%      │
├────────────────────────────────────────┼────────────┤
│ Other (ETS tables)                     │ 1.55%      │
├────────────────────────────────────────┼────────────┤
│ Metrics data                           │ 0.66%      │
├────────────────────────────────────────┼────────────┤
│ AMQP 0-9-1 channels                    │ 0.40%      │
├────────────────────────────────────────┼────────────┤
│ Message store indices                  │ 0.27%      │
├────────────────────────────────────────┼────────────┤
│ Atom table                             │ 0.24%      │
├────────────────────────────────────────┼────────────┤
│ Client connections: writer processes   │ 0.19%      │
├────────────────────────────────────────┼────────────┤
│ Quorum queue replica processes         │ 0.10%      │
├────────────────────────────────────────┼────────────┤
│ Stream replica processes               │ 0.07%      │
├────────────────────────────────────────┼────────────┤
│ Mnesia                                 │ 0.02%      │
├────────────────────────────────────────┼────────────┤
│ Metadata store                         │ 0.02%      │
├────────────────────────────────────────┼────────────┤
│ Stream coordinator processes           │ 0.02%      │
├────────────────────────────────────────┼────────────┤
│ Classic queue processes                │ 0.00%      │
├────────────────────────────────────────┼────────────┤
│ Metadata store ETS tables              │ 0.00%      │
├────────────────────────────────────────┼────────────┤
│ Stream replica reader processes        │ 0.00%      │
├────────────────────────────────────────┼────────────┤
│ Reserved by the kernel but unallocated │ 0.00%      │
└────────────────────────────────────────┴────────────┘
```

Note that there are [two different supported strategies](./memory-use#strategies)
for computing memory footprint of a node. `rabbitmqadmin` will use the greater value
for 100% when computing the relative share in percent for each category.

Other factors that can affect the precision of percentage values reported  are [runtime allocator](./memory-use#preallocated-memory)
behavior nuances and the [kernel page cache](./memory-use#page-cache).


### List feature flags and their state

```shell
rabbitmqadmin feature_flags list
```

```shell
# same command as above
rabbitmqadmin list feature_flags
```

### Enable a feature flag

```shell
rabbitmqadmin feature_flags enable rabbitmq_4.0.0
```

### Enable all stable feature flags

```shell
rabbitmqadmin feature_flags enable_all
```

### List deprecated features in use in the cluster

```shell
rabbitmqadmin deprecated_features list_used
```

### List all deprecated features

```shell
rabbitmqadmin deprecated_features list
```

```shell
# same command as above
rabbitmqadmin list deprecated_features
```

### Export Definitions

To export [definitions](./definitions) to standard output, use `definitions export --stdout`:

```shell
rabbitmqadmin definitions export --stdout
```

To export definitions to a file, use `definitions export --file /path/to/definitions.file.json`:

```shell
rabbitmqadmin definitions export --file /path/to/definitions.file.json
```

### Export and Transform Definitions

`definitions export` can transform the exported JSON definitions file it gets from the
target node. This is done by applying one or more transformations to the exported
JSON file.

This can be useful to remove classic queue mirroring-related keys (such as `ha-mode`) from a definitions
set originating from a 3.13.x node, or to obfuscate usernames and passwords, or exclude certain definitions file
sections entirely.

To specify what transformations should be applied, use the `--transformations` options,
which takes a comma-separated list of  supported operation names.

The following table explains what transformations are available and what they do:

| Transformation name            | Description                                                  |
|--------------------------------|--------------------------------------------------------------|
| `strip_cmq_keys_from_policies` | Deletes all classic queue mirroring-related keys (such as `ha-mode`) from all exported policies.<br/><br/>Must be followed by `drop_empty_policies` to strip off the policies whose definition has become empty (and thus invalid at import time) after the removal of all classic queue mirroring-related keys |
| `drop_empty_policies`          | Should be used after `strip_cmq_keys_from_policies` to strip off the policies whose definition has become empty (and thus invalid at import time) after the removal of all classic queue mirroring-related keys |
| `obfuscate_usernames`          | Replaces usernames and passwords with dummy values.<br/><br/>For usernames the values used are: `obfuscated-username-1`, `obfuscated-username-2`, and so on.<br/><br/>For passwords the values generated are: `password-1`, `password-2`, and so forth.<br/><br/>This transformations updates both the users and the permissions sections, consistently |
| `exclude_users`                | Removes all users from the result. Commonly used together with `exclude_permissions` |
| `exclude_permissions`          | Removes all permissions from the result. Commonly used together with `exclude_users` |
| `exclude_runtime_parameters`   | Removes all runtime parameters (including federation upstreams, shovels, WSR and SDS settings in Tanzu RabbitMQ) from the result |
| `exclude_policies`             | Removes all policies from the result                         |
| `no_op`                        | Does nothing. Can be used as the default in dynamically computed transformation lists, e.g. in scripts |


The following command applies two transformations named `strip_cmq_keys_from_policies` and `drop_empty_policies`
that will strip all classic queue mirroring-related policy keys that RabbitMQ 3.13 nodes supported,
then removes the policies that did not have any keys left (ended up having an empty definition):

```shell
# strips classic mirrored queue-related policy keys from the exported definitions, then prints them
# to the standard output stream
rabbitmqadmin definitions export --stdout --transformations strip_cmq_keys_from_policies,drop_empty_policies
```

The following example exports definitions without users and permissions:

```shell
# removes users and user permissions from the exported definitions, then prints them
# to the standard output stream
rabbitmqadmin definitions export --stdout --transformations exclude_users,exclude_permissions
```

To export definitions with usernames replaced by dummy values (usernames: `obfuscated-username-1`, `obfuscated-username-2`, and so on;
passwords: `password-1`, `password-2`, and so forth), use the `obfuscate_usernames` transformation:

```shell
rabbitmqadmin definitions export --file /path/to/definitions.file.json --transformations obfuscate_usernames
```

### Import Definition

To import definitions from the standard input, use `definitions import --stdin`:

```shell
cat /path/to/definitions.file.json | rabbitmqadmin definitions import --stdin
```

To import definitions from a file, use `definitions import --file /path/to/definitions.file.json`:

```shell
rabbitmqadmin definitions import --file /path/to/definitions.file.json
```

### Declare an AMQP 0-9-1 Shovel

To declare a [dynamic shovel](./shovel-dynamic) that uses AMQP 0-9-1 for both source and desitnation, use
`shovels declare_amqp091`:

```shell
rabbitmqadmin shovels declare_amqp091 --name my-amqp091-shovel \
    --source-uri amqp://username:s3KrE7@source.hostname:5672 \
    --destination-uri amqp://username:s3KrE7@source.hostname:5672 \
    --ack-mode "on-confirm" \
    --source-queue "src.queue" \
    --destination-queue "dest.queue" \
    --predeclared-source false \
    --predeclared-destination false
```

### Declare an AMQP 1.0 Shovel

To declare a [dynamic shovel](./shovel-dynamic) that uses AMQP 1.0 for both source and desitnation, use
`shovel declare_amqp10`.

Note that

1. With AMQP 1.0 shovels, credentials in the URI are mandatory (there are no defaults)
2. With AMQP 1.0 shovels, the topology must be pre-declared (an equivalent of `--predeclared-source true` and `--predeclared-destination true` for AMQP 0-9-1 shovels)
3. AMQP 1.0 shovels should use [AMQP 1.0 addresses v2](./amqp#addresses)

```shell
rabbitmqadmin shovels declare_amqp10 --name my-amqp1.0-shovel \
    --source-uri "amqp://username:s3KrE7@source.hostname:5672?hostname=vhost:src-vhost" \
    --destination-uri "amqp://username:s3KrE7@source.hostname:5672?hostname=vhost:dest-vhost" \
    --ack-mode "on-confirm" \
    --source-address "/queues/src.queue" \
    --destination-address "/queues/dest.queue"
```

### List Shovels

To list shovels across all virtual hosts, use `shovels list_all`:

```shell
rabbitmqadmin shovels list_all
```

### Delete a Shovel

To delete a shovel, use `shovels delete --name`:

```shell
rabbitmqadmin shovels delete --name my-amqp091-shovel
```

### List Federation Upstreams

To list [federation upstreams](./federation) across all virtual hosts, use `federation list_all_upstreams`:

```shell
rabbitmqadmin federation list_all_upstreams
```

### Create a Federation Upstream for Exchange Federation

To create a [federation upstream](./federated-exchanges), use `federation declare_upstream_for_exchanges`.
This command provides a reduced set of options, only those that are relevant
specifically to exchange federation.

```shell
rabbitmqadmin --vhost "local-vhost" federation declare_upstream_for_exchanges --name "pollux" \
                --uri "amqp://pollux.eng.megacorp.local:5672/remote-vhost" \
                --ack-mode 'on-publish' \
                --prefetch-count 2000 \
                --exchange-name "overridden.name" \
                --queue-type quorum \
                --bind-using-nowait true
```

### Create a Federation Upstream for Queue Federation

To create a [federation upstream](./federated-queues), use `declare_upstream_for_queues`.
This command provides a reduced set of options, only those that are relevant
specifically to queue federation.

```shell
rabbitmqadmin --vhost "local-vhost" federation declare_upstream_for_queues --name "clusters.sirius" \
                --uri "amqp://sirius.eng.megacorp.local:5672/remote-vhost" \
                --ack-mode 'on-publish' \
                --prefetch-count 2000 \
                --queue-name "overridden.name" \
                --consumer-tag "overriden.ctag"
```

### Create a Universal Federation Upstream

To create a [federation upstream](./federation) that will be (or can be)
used for federating both queues and exchanges, use `declare_upstream`. It combines
[all the federation options](./federation-reference), that is,
the options of both `declare_upstream_for_queues` and `declare_upstream_for_exchanges`.

```shell
rabbitmqadmin --vhost "local-vhost" federation declare_upstream --name "pollux" \
                --uri "amqp://pollux.eng.megacorp.local:5672/remove-vhost" \
                --ack-mode 'on-publish' \
                --prefetch-count 2000 \
                --queue-name "overridden.name" \
                --consumer-tag "overriden.ctag" \
                --exchange-name "overridden.name" \
                --queue-type quorum \
                --bind-using-nowait true
```

### Delete a Federation Upstream

To delete a [federation upstream](./federation), use 'federation delete_upstream',
which takes a virtual host and an upstream name:

```shell
rabbitmqadmin --vhost "local-vhost" federation delete_upstream --name "upstream.to.delete"
```

### List Federation Links

To list all [federation links](./federation) across all virtual hosts, use `federation list_all_links`:

```shell
rabbitmqadmin federation list_all_links
```

### Rebalance Queue Leaders

To [rebalance quorum queue](./quorum-queues#member-rebalancing) leaders across cluster nodes, use `rebalance queues`. This operation helps distribute queue leaders more evenly across cluster nodes, which can more evenly distribute load and improve resource utilization, depending on the workload.


```shell
# Rebalance queue leaders in the default virtual host (/)
rabbitmqadmin rebalance queues
```

```shell
# Rebalance queue leaders in a specific virtual host
rabbitmqadmin --vhost "production" rebalance queues
```

This operation is asynchronous and may take time to complete depending on the number of queues in the cluster. Monitor RabbitMQ logs for completion status.


### Inspect Target Endpoint (for Troubleshooting)

Use `show endpoint` to display the computed HTTP API endpoint URI.


```shell
rabbitmqadmin show endpoint
```

This command helps verify that `rabbitmqadmin` is targeting the correct RabbitMQ HTTP API endpoint, for example, when a `rabbitmqadmin.conf` file exists or environment variables may be set in the environment.

### Health Checks

`rabbitmqadmin` provides various [health check](./monitoring#health-checks) commands:


```shell
# Check for local resource alarms on the target node
rabbitmqadmin health_check local_alarms
```

```shell
# Check for resource alarms across the entire cluster
rabbitmqadmin health_check cluster_wide_alarms
```

```shell
# Check if node is quorum critical (queues/streams would lose quorum if node shuts down)
rabbitmqadmin health_check node_is_quorum_critical
```

```shell
# Check for deprecated features in use across the cluster
rabbitmqadmin health_check deprecated_features_in_use
```

```shell
# Verify TCP listener is reachable on specific port
rabbitmqadmin health_check port_listener --port 5672
```

```shell
# Verify TCP listener is reachable for specific protocol
rabbitmqadmin health_check protocol_listener --protocol amqp091
```

These health checks are useful for monitoring scripts and automated cluster health verification.

### TLS Configuration

`rabbitmqadmin` supports connecting to RabbitMQ clusters that use [TLS](./ssl) for the HTTP API:


```shell
# Connect using TLS with default settings
rabbitmqadmin --use-tls show overview
```

```shell
# Connect using TLS with custom CA certificate bundle
rabbitmqadmin --use-tls --tls-ca-cert-file /path/to/ca-bundle.pem show overview
```

```shell
# Connect using TLS with client certificate authentication
rabbitmqadmin --use-tls \
  --tls-ca-cert-file /path/to/ca-bundle.pem \
  --tls-cert-file /path/to/client-cert.pem \
  --tls-key-file /path/to/client-key.pem \
  show overview
```

```shell
# Disable TLS peer verification (not recommended for production)
rabbitmqadmin --use-tls --disable-tls-peer-verification show overview
```

### Password Hashing

`rabbitmqadmin` can generate password hashes compatible with RabbitMQ's [password hashing](./passwords) system:


```shell
# Hash a password using SHA256 (default)
rabbitmqadmin passwords salt_and_hash "my-secret-password"
```

```shell
# Override --hashing-algorithm if RabbitMQ nodes are configured to use SHA512
rabbitmqadmin passwords salt_and_hash "my-secret-password" --hashing-algorithm SHA512
```

This is useful for pre-computing password hashes for user management scripts or when working with RabbitMQ definitions files.

### Listing policies

```shell
rabbitmqadmin list policies
```

```shell
# List policies in a specific virtual host
rabbitmqadmin --vhost "events" policies list_in
```

### Create a Policy

```shell
# Create a queue policy for high availability
rabbitmqadmin --vhost "events" policies declare --name "ha-all" --pattern ".*" --definition '{"ha-mode":"all","ha-sync-mode":"automatic"}' --priority 1
```

```shell
# Create a TTL policy for temporary queues
rabbitmqadmin --vhost "events" policies declare --name "temp-queues" --pattern "temp\\..*" --definition '{"message-ttl":300000,"expires":600000}' --priority 2
```

### Delete a Policy

```shell
rabbitmqadmin --vhost "events" policies delete --name "ha-all"
```

### List policies matching an object

```shell
# Show which policies apply to a specific queue
rabbitmqadmin --vhost "events" policies list_matching_object --name "my.queue" --type "queue"
```

### Listing operator policies

```shell
rabbitmqadmin list operator_policies
```

```shell
# List operator policies in a specific virtual host
rabbitmqadmin --vhost "events" operator_policies list_in
```

### Create an Operator Policy

```shell
# Create an operator policy to set max queue length
rabbitmqadmin --vhost "events" operator_policies declare --name "queue.max-length" --pattern ".*" --definition '{"max-length":10000}' --priority 1
```

### Delete an Operator Policy

```shell
rabbitmqadmin --vhost "events" operator_policies delete --name "queue.max-length"
```

### Listing runtime parameters

```shell
rabbitmqadmin list parameters
```

```shell
# List parameters in a specific virtual host
rabbitmqadmin --vhost "events" parameters list
```

### Listing global parameters

```shell
rabbitmqadmin list global_parameters
```


## Subcommand and Long Option Inference

This feature is available only in the `main` branch
at the moment.

If the `RABBITMQADMIN_NON_INTERACTIVE_MODE` is not set to `true`, this tool
now can infer subcommand and --long-option names.

This means that a subcommand can be referenced with its unique prefix,
that is,

* 'queues del' will be inferred as 'queues delete'
* 'q del --nam "a.queue"' will be inferred as 'queues delete --name "a.queue"'

To enable each feature, set the following environment variables to
'true':

* `RABBITMQADMIN_INFER_SUBCOMMANDS`
* `RABBITMQADMIN_INFER_LONG_OPTIONS`

This feature is only meant to be used interactively. For non-interactive
use, it can be potentially too dangerous to allow.


## Configuration Files

`rabbitmqadmin` v2 supports [TOML](https://toml.io/en/)-based configuration files
stores groups of HTTP API connection settings under aliases ("node names" in original `rabbitmqadmin` speak).

Here is an example `rabbitmqadmin` v2 configuration file:

```toml
[local]
hostname = "localhost"
port = 15672
username = "lolz"
password = "lolz"
vhost = '/'

[staging]
hostname = "192.168.20.31"
port = 15672
username = "staging-2387a72329"
password = "staging-1d20cfbd9d"

[production]
hostname = "(redacted)"
port = 15671
username = "user-2ca6bae15ff6b79e92"
password = "user-92ee4c479ae604cc72"
```

Instead of specifying `--hostname` or `--username` on the command line to connect to
a cluster (or specific node) called `staging`, a `--node` alias can be specified instead:

```shell
# will use the settings from the section called [staging]
rabbitmqadmin --node staging show churn
```

Default configuration file path is at `$HOME/.rabbitmqadmin.conf`, as it was in
the original version of `rabbitmqadmin`. It can be overridden on the command line:

```shell
# will use the settings from the section called [staging]
rabbitmqadmin --config $HOME/.configuration/rabbitmqadmin.conf --node staging show churn
```

## Breaking or Potentially Breaking Changes Compared to v1

### Some Non-Essential Features Were Dropped

`rabbitmqadmin` v2 does not support

 * Sorting of results. Instead, use `--non-interactive` and parse the spaces-separated
   output. Many modern tools for working with data parse it into a table, sort the data set,
   filter the results, and son. In fact, these features for data processing are ready available [in some shells](https://www.nushell.sh/)
 * Column selection. This feature may be reintroduced
 * JSON output for arbitrary commands (with the exception of `definitions` commands).
   Use the [HTTP API directly](./management#http-api) if you need to work with JSON
 * CSV output for arbitrary commands. This format may be reintroduced

### --snake-case for Command Options

`rabbitmqadmin` v1 used `lower_case` for named command arguments, for example:

```shell
# Note: auto_delete
rabbitmqadmin-v1 --vhost "vh-2" queues declare name="qq.1" type="quorum" durable=true auto_delete=false
```

`rabbitmqadmin` v2 uses a more typical `--snake-case` format for the same arguments:

```shell
# Note: --auto-delete
rabbitmqadmin --vhost "vh-2" queues declare --name "qq.1" --type "quorum" --durable true --auto-delete false
```

### Global Arguments Come First

Global flags in `rabbitmqadmin` v2 must precede the command category (e.g. `list`) and the command itself,
namely various HTTP API endpoint options and `--vhost`:

```shell
rabbitmqadmin --vhost "events" queues declare --name "target.quorum.queue.name" --type "quorum" --durable true
```

### --prefix Overrides API Path Prefix

In `rabbitmqadmin` v1, `--path-prefix` appended to the default [API path prefix](./management#path-prefix).
In this version, the value passed to `--path-prefix` will be used as given, in other words,
it replaces the default prefix, `/api`.

### Configuration File Format Moved to TOML

`rabbitmqadmin` v1 supported ini configuration files that allowed
the user to group a number of command line values under a name, e.g. a cluster or node nickname.

Due to the "no dependencies other than Python" design goal of `rabbitmqadmin` v1, this feature was not really tested,
and the specific syntax (that of ini files, supported by Python's [`ConfigParser`](https://docs.python.org/3/library/configparser.html)) linting, parsing or generation tools were not really available.

`rabbitmqadmin` v2 replaces this format with [TOML](https://toml.io/en/), a popular configuration standard
with [verification and linting tools](https://www.toml-lint.com/), as well as very mature parser
that is not at all specific to `rabbitmqadmin` v2.

Here is an example `rabbitmqadmin` v2 configuration file:

```toml
[local]
hostname = "localhost"
port = 15672
username = "lolz"
password = "lolz"
vhost = '/'

[staging]
hostname = "192.168.20.31"
port = 15672
username = "staging-2387a72329"
password = "staging-1d20cfbd9d"

[production]
hostname = "(redacted)"
port = 15671
username = "user-efe1f4d763f6"
password = "(redacted)"
```


## `rabbitmqadmin` v1

`rabbitmqadmin` v1 is the original CLI tool for the HTTP API, historically developed
as part of the [management plugin](./management/) and distributed with it.

It is no longer under active development.

### Obtaining `rabbitmqadmin` v1

:::important

Consider switching to `rabbitmqadmin` v2. The original `rabbitmqadmin` is no longer under
active development.

:::

`rabbitmqadmin` v1 can be downloaded from any RabbitMQ node that has
the management plugin enabled. Navigate to `http://{hostname}:15672/cli/rabbitmqadmin` to download it.
The tool requires a supported version of Python to be installed.

Alternatively, `rabbitmqadmin` v1 can be downloaded [from GitHub](https://raw.githubusercontent.com/rabbitmq/rabbitmq-server/v4.1.x/deps/rabbitmq_management/bin/rabbitmqadmin).
