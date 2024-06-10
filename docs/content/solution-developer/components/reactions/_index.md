---
type: "docs"
title: "Reactions"
linkTitle: "Reactions"
weight: 60
description: >
    What are Reactions and How to Use Them
---

Reactions process the stream of query result changes output by one or more Continuous Queries and act on them. 

{{< figure src="simple-end-to-end.png" alt="End to End" width="65%" >}}

The action taken depends on the Reaction type being used. Drasi currently provides the following Reactions:

- [Azure Event Grid](#event-grid-reaction), to forward Continuous Query results to Azure Event Grid, which in turn enables integration with any application, service, or function that can receive updates from Azure Event Grid.
- [Debug](#debug-reaction), a tool to help developers inspect the results generated by Continuous Queries.
- [SignalR](#signalr-reaction), to forward Continuos Query results to Web Applications.
- [Gremlin](#gremlin-reaction), to use the Continuous Query results as parameters to commands that run against a Gremlin database.

*To create custom Reactions, see the [Custom Reactions](/platform-developer/reactions/) section of the [Platform Developer Guide](/platform-developer)*

## Creation
Reactions can be created and managed using the `drasi` CLI tool. 

The easiest way to create a Reaction, and the way you will often create one as part of a broader software solution, is to:

1. Collect ID's of the Continuous Queries the Reaction will subscribe to.
1. Collect credentials and endpoint addresses that provide access to any external system the Reaction interacts with.
1. Create a YAML file containing the Reaction Resource Definition. This will include the configuration settings that enable the Reaction to connect to external systems. This can be stored in your solution repo and versioned along with all the other solution code / resources.
1. Run `drasi apply` to apply the Reaction resource definition to the Kubernetes cluster where your Drasi environment is deployed.

As soon as the Reaction is created it will start running, subscribing to its Continuous Queries and processing query result changes.

The definition for a Reaction has the following structure:

```
apiVersion: v1
kind: Reaction
name: <reaction-id>
spec:
  image: <reaction-image>
  queries:
    query1: <custom metadata for query1 (optional)>
    query2: <custom metadata for query2 (optional)>
  properties:
    <property_1_name>: <property_1_value>
    <property_2_name>: 
      kind: Secret
      name: <secret_id>
      key: <secret_key>          
  endpoints:
    <endpoint_name>: <enpoint_port_num>
```

The following table provides a summary of these configuration settings:

|Name|Description|
|-|-|
|apiVersion|Must have the value **v1**|
|kind|Must have the value **Reaction**|
|name|The **id** of the Reaction. Must be unique within the scope of the Reactions in the Drasi deployment. The  **id** is used to manage the Reaction.|
|spec.image|The name of the docker image for the Reaction.|
|spec.queries|The list of Continuous Query IDs the Reaction will subscribe to. Some Reactions also need per-query configuration, which can be passed using the options property of the queryId. These are unique to the type of Reaction and are detailed in the sections below.|
|spec.properties|Name/value pairs used to configure the Reaction. These are unique to the type of Reaction and are detailed in the sections below.|
|spec.endpoints|Names and port numbers to use for Reactions that expose accessible ports for clients to connect to.|  

Once configured, to create a Reaction defined in a file called `reaction.yaml`, you would run the command:

```
drasi apply -f reaction.yaml
```

You can then use additional `drasi` commands to query the existence and status of the Reaction resource. For example, to see a list of the active Reactions, run the following command:

```
drasi list reaction
```

## Deletion
To delete an active Reaction, run the following command:

```
drasi delete reaction <reaction-id>
```

For example, if the Reaction ID is `update-gremlin`, you would run,

```
drasi delete reaction update-gremlin
```

## Configuring Reactions
The following sections describe the configuration of the Reaction types currently supported by Drasi.

### Azure Event Grid Reaction
The Event Grid Reaction requires the following configuration settings:

|Name|Type|Description|
|-|-|-|
|image| | Must have the value **reaction-eventgrid**|
|EventGridUri| Property | |
|EventGridKey| Property | |

The following is an example of a fully configured Event Grid Reaction using Kubernetes Secrets to securely store sensitive information:

```bash
kubectl create secret generic credentials --from-literal=access-key=xxxxxx
```

```
apiVersion: v1
kind: Reaction
name: eventgrid1
spec:
  image: reaction-eventgrid
  EventGridUri: https://reactive-graph-daniel.westus-1.eventgrid.azure.net/api/events
  EventGridKey: 
    kind: Secret
    name: credentials
    key: access-key      
  queries:
    my-query1:
```

### Gremlin Reaction
TODO

### SignalR Reaction
The SignalR Reaction requires the following configuration settings:

|Name|Type|Description|
|-|-|-|
|image| | Must have the value **reaction-signalr**|
|AzureSignalRConnectionString| Property | |
|gateway| Endpoint | |

The following is an example of a fully configured Event Grid Reaction using Kubernetes Secrets to securely store sensitive information:

```bash
kubectl create secret generic credentials --from-literal=connection-string=xxxxxx
```

```
apiVersion: v1
kind: Reaction
name: signalr1
spec:
  image: reaction-signalr
  AzureSignalRConnectionString:
    kind: Secret
    name: credentials
    key: connection-string          
  endpoints:
    gateway: 8080
  queries:
    my-query1:
```

### Debug Reaction
TODO

## Creating a new Reaction
You can develop custom reactions by writing an application in any language that adheres to a certain specification and publish it as a docker image to the registry serving Drasi images to your cluster.

### Query Configuration

The Drasi control plane will mount a folder at `/etc/queries` where each file will be named after each queryId that is configured for the reaction.  The contents of each file will be the `options` field from the config.

Consider the following reaction configuration, this will result in 3 empty files named `query1`, `query2` and `query3` under `/etc/queries`.

```yaml {hl_lines=["7-11"]}
apiVersion: query.reactive-graph.io/v1
kind: Reaction
metadata:
  name: my-reaction
spec:
  reactionImage: my-reaction
  queries:
    - queryId: query1
    - queryId: query2
    - queryId: query3
```

The following reaction configuration shows how to include additional metadata per query.  This will result in a file named `query1` with the contents of `foo` and a file named `query2` with the contents of `bar`

```yaml{hl_lines=["9-10","12-13"]}
apiVersion: query.reactive-graph.io/v1
kind: Reaction
metadata:
  name: my-reaction
spec:
  reactionImage: my-reaction
  queries:
    - queryId: query1
      options: >
      foo
    - queryId: query2
      options: >
      bar
```

The format of the content of the options field is completely up to the developer of that particular reaction, for example you could include yaml or json content and it is up to you to deserialize and make sense of it within the context of your custom reaction.

### Receiving Changes

When the projection of a continuous query is changed, a message will be published to a [Dapr topic](https://docs.dapr.io/developing-applications/building-blocks/pubsub/howto-publish-subscribe/#subscribe-to-topics). The pubsub name will be available on the `PUBSUB` environment variable (default is `rg-pubsub`). The topic name will be `<queryId>-results`, so for each queryId you discover in `/etc/queries`, you should subscribe to that Dapr topic.

A skeleton implementation in Javascript would look something like this

```js
import { DaprServer } from "@dapr/dapr";
import { readdirSync, readFileSync } from 'fs';
import path from 'path';

const pubsubName = process.env["PUBSUB"] ?? "rg-pubsub";
const configDirectory = process.env["QueryConfigPath"] ?? "/etc/queries";
const daprServer = new DaprServer();

let queryIds = readdirSync(configDir);
  for (let queryId of queryIds) {
    if (!queryId || queryId.startsWith("."))
      continue;

    await daprServer.pubsub.subscribe(pubsubName, queryId + "-results", (events) => {
        //implement code that reacts to changes here
    });
}

await daprServer.start();
```

### Message Format

The format of the incoming messages is a Json array, with each item itself containing an array for `addedResults`, `deletedResults` and `updatedResults`.

The basic structure looks like this

```json
[
    {
        "addedResults": [],
        "deletedResults": [],
        "updatedResults": [],
        "metadata": {}
    }
]
```

An example of a row being added to the continuous query projection would look like this

```json {hl_lines=["3-8"]}
[
    {
        "addedResults": [
            {
                "Id": 1,
                "Name": "Foo"
            }
        ],
        "deletedResults": [],
        "updatedResults": []
    }
]
```

An example that row being updated would look like this

```json {hl_lines=["5-16"]}
[
    {
        "addedResults": [],
        "deletedResults": [],
        "updatedResults": [
            {
                "before": {
                    "Id": 1,
                    "Name": "Foo"
                },
                "after": {
                    "Id": 1,
                    "Name": "Bar"
                }
            }
        ]
    }
]
```

An example that row being deleted would look like this

```json {hl_lines=["4-9"]}
[
    {
        "addedResults": [],
        "deletedResults": [
            {
                "Id": 1,
                "Name": "Bar"
            }
        ],
        "updatedResults": []
    }
]
```


## Registering a new reaction
To add support for a new kind of Reaction to Drasi, you must develop the services that will connect to the Reaction and register a new Reaction Provider with Drasi. The Reaction Provider definition describes the services Drasi must run, where to get the images, and the configuration settings that are required when an instance of that Reaction is created

The definition for a ReactionProvider has the following basic structure:

```yaml
apiVersion: v1
kind: ReactionProvider
name: <name>
tag: <tag>   # Optional.
spec:
  services:
    <reaction-name>: 
        image: <image_name> # Required. Cannot be overwritten.
        dapr: # Optional; used for specifying dapr-related annotations
            app-port: <value> # Optional
            app-protocol: <value> # Optional
        endpoints: # Optional; used for configuring internal/external endpoints
            <endpoint_name>:
                setting: internal/external
                target: <target>  # name of the config to use, which
                                  # should be defined under the 
                                  # `config_schema` section of the service
            (any additional endpoints)...
        config_schema: # Optional; used for specifying any additional environment variables
            type: object
            properties: 
                <name>:
                    type: <type>  # One of [string, integer, boolean, array or object]
                    default: <value> # Optional.
                (any additioanl properties)...
            required: # Optional. List any required properties here
  config_schema: # Optional; 
                 # The environment variables defined here will be 
                 # accessible by all services
    type: object
    properties: 
        <name>:
            type: <type>  # One of [string, integer, boolean, array or object]
            default: <value> # Optional.
        (any additioanl properties)...
    required: # Optional. List any required properties here
    
```
In the ReactionProvider definition:
- **apiVersion**: Must be **v1**
- **kind**: Must be **ReactionProvider**
- **name**: Specifies the kind of Reaction that we are trying to create
- **tag**: Optional. This is used for specifying the "version" of the ReactionProvider


The section below provides a more detailed walkthrough of the various fields under the `spec` section.
### Config Schema

The `config_schema` section that is at the top level is used for defining any enviroment variables that will be shared and accessible by all services. Similarly, this field can be defined in a similar way as how you would define the `config_schema` field for each service.

For example, the following section will specify two environment variables `foo` and `isTrue` for this Reaction. `foo` is a required environment variable and it expects the input to be of type `string`, whereas `isTrue` expects the input to be of type `boolean` and is not a required value (default value is set to `true`)

```yaml
spec:
   services:
     ...
   config_schema:
      type: object
      properties:
        foo:
          type: string
        isTrue:
          type: boolean
          default: true
      required:
        - foo
```


### Services

The `services` field configures the definition of the serivce(s) of a Reaction. At the moment, a service must be defined for any ReactionProvider. For each `service`, there are four fields that you can configure:
- `image`
  - `image` is a required field and you can specify the image to use for this Reaction service here. 
    - (NOTE: Drasi assumes that the image lives in the same registry that you used when you executed `drasi init`).
  - `endpoints`
    - If your Reaction has a port that needs to be exposed, you can specify them under the `endpoints` section. The `endpoints` section takes in a series of `endpoint`, which is a JSON object. Each `endpoint` object should have two properties: `setting` and `target`. `setting` can be either "internal" or "external", althrough we currently only support internal endpoints. The `target` field will reference the value of a config that is defined under the `config_schema` section of the service. You can provide a default value when defining the ReactionProvider and/or overwrite in the actual Reaction definition file.
    - Each endpoint will be rendered into a Kuberentes Service, with the value of `target` being set as the port number.
    - The following block defines a Reaction that will create a Kubernetes service called `<Reaction-name>-gateway` with a default port of `4318` when deployed.
      - ```yaml 
          endpoints:
            gateway:
              setting: internal
              target: gateway-port
          config_schema:
            type: object
            gateway-port:
              type: number
              default: 4318
  - `dapr`: optional. This field is used for specifying any [dapr annotation](https://docs.dapr.io/reference/arguments-annotations-overview/) that the user wishes to include. Currently we only support `app-port` and `app-protocol`. 
    - The `app-port` annotation is used to tell Dapr which port the application is listening on, whereas the `app-protocol` annotation configures the protocol that Dapr uses to communicate with your app
      - Sample yaml block:
      - ```yaml 
          dapr:
            app-port: 4002
  - `config_schema`
    - This is used for defining environment variables; however, the environment variables that are defined here are only accessible for this particular service.
    - The configurations are defined by following JSON Schema. We define this field to be of type `object`, and list all of the configs (environment variables) under the `properties` section. For each of the property, you need to specify its type and an optional default value. For any required environment variables, you can list them under the `require` section as an array of elements
    - Sample:
     ```yaml
        config_schema:
          type: object
          properties:
            foo:
              type: string
              default: bar
            property2:
              type: boolean
              default: true
          required:
            - foo


### Validating the ReactionProvider file
To validate a ReactionProvider yaml file, there are two approaches:
1. Using `apply` command from the Drasi CLI. The CLI will automatically validate the ReactionProvider before registering it. 
2. Using the [Drasi VSCode Extension](/solution-developer/vscode-extension/). The extension will detect all of the ReactionProvider yaml files in the current workspace. Click on the `Validate` button next to each instance to validate a specific ReactionProvider definition.

### Sample ReactionProvider and Reaction file
This section contains a sample Reactionprovider file and Reaction file for the `Debug` reaction. 
The `Debug` reaction:
- Only needs one service with the name `debug`.
- Uses the `reaction-debug` image
- Needs to have an internal (k8s) endpoint at port "8080"
- `tag` should be `v1`

ReactionProvider file:
```yaml
apiVersion: v1
kind: ReactionProvider
name: Debug
spec:
  queries:
    type: objects
  services:
    debug:
      image: reaction-debug
      endpoints:
        gateway:
          setting: internal
          target: port
      config_schema:
        type: object
        properties:
          port:
            type: number
            default: 8080
```

The ReactionProvider can be applied using the Drasi CLI: 
```
drasi apply -f <name-of-the-provider-file>.yaml
```


Reaction file:
```yaml
apiVersion: v1
kind: Reaction
name: hello-world-debug
spec:
  kind: Debug:v1
  queries:
    hello-world-from:
    message-count: 
    inactive-people: 
```
Similarly, this Reaction file can also be applied using the CLI:
```
drasi apply -f <name-of-the-reaction-file>.yaml
```