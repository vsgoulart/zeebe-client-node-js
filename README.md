# Zeebe Node.js Client

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
![Test on Camunda Cloud](https://github.com/jwulf/zeebe-client-node-js/workflows/Test%20on%20Camunda%20Cloud/badge.svg)

This is a Node.js gRPC client for [Zeebe](https://zeebe.io). It is written in TypeScript and transpiled to JavaScript in the `dist` directory.

Comprehensive API documentation is available [online](https://github.com/zeebe-io/zeebe-client-node-js).

See [CHANGELOG.md](./CHANGELOG.md) to see what has changed with each release.

Docker-compose configurations for Zeebe are available at [zeebe-docker-compose](https://github.com/zeebe-io/zeebe-docker-compose).

Get a hosted instance of Zeebe on [Camunda Cloud](https://camunda.io).

## Table of Contents

-   [ Versioning ](#versioning)
-   [ gRPC Implementation ](#grpc-implementation)
-   [ Type difference from other Zeebe clients ](#type-difference)
-   [ A note on representing timeout durations ](#time-duration)

**Quick Start**

-   [ Install ](#install)
-   [ Get Broker Topology ](#get-topology)
-   [ Deploy a workflow ](#deploy-workflow)

**Connection Behaviour**

-   [ Client-side gRPC retry in ZBClient ](#client-side-retry)
-   [ onReady(), onConnectionError(), and connected ](#on-ready)
-   [ Initial Connection Tolerance ](#initial-connection-tolerance)

**Connecting to a Broker**

-   [ TLS ](#tls)
-   [ OAuth ](#oauth)
-   [ Basic Auth ](#basic-auth)
-   [ Camunda Cloud ](#camunda-cloud)
-   [ Zero-conf constructor ](#zero-conf)

**Job Workers**

-   [ Job Workers](#job-workers)
-   [ The `ZBWorker` Job Worker ](#create-zbworker)
-   [ Unhandled Exceptions in Task Handlers ](#unhandled-exceptions)
-   [ Completing tasks with success, failure, error, or forwarded ](#complete-tasks)
-   [ Working with Workflow Variables and Custom Headers ](#working-with-variables)
-   [ Constraining the Variables Fetched by the Worker ](#fetch-variable)
-   [ The "Decoupled Job Completion" pattern ](#decoupled-complete)
-   [ The `ZBBatchWorker` Job Worker ](#zbbatchworker)
-   [ Long polling ](#long-polling)

**Client Commands**

-   [ Start a Workflow Instance ](#start-workflow)
-   [ Start a Workflow Instance of a specific version of a Workflow definition ](#start-specific-version)
-   [ Start a workflow instance and await the workflow outcome ](#start-await)
-   [ Publish a Message ](#publish-message)
-   [ Publish a Start Message ](#publish-start-message)
-   [ Activate Jobs ](#activate-jobs)

**Other Concerns**

-   [ Graceful Shutdown ](#graceful-shutdown)
-   [ Logging ](#logging)

**Programming with Safety**

-   [ Generating TypeScript constants for BPMN Models ](#generate-constants)
-   [ Generating code from a BPM Model file ](#generate-code)
-   [ Writing Strongly-typed Job Workers ](#strongly-typed)
-   [ Run-time Type Safety ](#run-time-safety)

**Development of the Library itself**

-   [ Developing Zeebe Node ](#developing)
    -   [ Tests ](#tests)
    -   [ Writing Tests ](#writing-tests)
-   [ Contributors ](#contributors)

<a name = "versioning"></a>

## Versioning

To enable that the client libraries can be easily supported to the Zeebe server we map the version numbers, so that Major, Minor match the server application. Patches are independent and indicate client updates.

NPM Package version 0.24.x supports Zeebe 0.22.x and above

NPM Package version 0.23.x supports Zeebe 0.22.x and above

NPM Package version 0.22.x supports Zeebe 0.22.x

NPM Package version 0.21.x supports Zeebe 0.21.x

<a name="grpc-implementation"></a>

## gRPC Implementation

The Zeebe Node client uses the C gRPC client implementation [grpc-node](https://github.com/grpc/grpc-node) by default.

From 0.23.3, you can opt-in to use the pure JavaScript implementation [@grpc/grpc-js](https://www.npmjs.com/package/@grpc/grpc-js), to assist us in testing it.

To activate the pure JavaScript implementation, set `ZEEBE_NODE_PUREJS=true`.

Please report any issues in [GitHub](https://github.com/zeebe-io/zeebe-client-node-js/issues).

<a name = "type-difference"></a>

## Type difference from other Zeebe clients

Protobuf fields of type `int64` are serialised as type string in the Node library. These fields are serialised as numbers (long) in the Go and Java client. See [grpc/#7229](https://github.com/grpc/grpc/issues/7229) for why the Node library serialises them as string. The Workflow instance key, and other fields that are of type long in other client libraries, are type string in this library. Fields of type `int32` are serialised as type number in the Node library.

<a name = "time-duration"></a>

## A note on representing timeout durations

All timeouts are ultimately communicated in _milliseconds_. They can be specified using the primitive type `number`, and this is always a _number of milliseconds_.

All timeouts in the client library can _also_, optionally, be specified by a time value that encodes the units, using the [typed-durations](https://www.npmjs.com/package/typed-duration) package. You can specify durations for timeouts like this:

```
const { Duration } = require('zeebe-node')

const timeoutS = Duration.seconds.of(30) // 30s timeout
const timeoutMs = Duration.milliseconds.of(30000) // 30s timeout in milliseconds
```

Using the value types makes your code more semantically specific.

There are four timeouts to take into account.

The first is the job `timeout`. This is the amount of time that the broker allocates exclusive responsibility for a job to a worker instance. By default, this is 60 seconds. This is the default value set by this client library. See "[Job Workers](#job-workers)".

The second is the `requestTimeout`. Whenever the client library sends a gRPC command to the broker, it has an explicit or implied `requestTimeout`. This is the amount of time that the gRPC gateway will wait for a response from the broker cluster before returning a `4 DEADLINE` gRPC error response.

If no `requestTimeout` is specified, then the configured timeout of the broker gateway is used. Out of the box, this is 15 seconds by default.

The most significant use of the `requestTimeout` is when using the `createWorkflowInstanceWithResult` command. If your workflow will take longer than 15 seconds to complete, you should specify a `requestTimeout`. See "[Start a Workflow Instance and await the Workflow Outcome](#start-await)".

The third is the `longpoll` duration. This is the amount of time that the job worker holds a long poll request to activate jobs open.

The final one is the maximum back-off delay in client-side gRPC command retries. See "[Client-side gRPC retry in ZBClient](#client-side-retry)".

## Quick Start

<a name = "install"></a>

## Install

### Add the Library to your Project

```bash
npm i zeebe-node
```

<a name = "get-topology"></a>

### Get Broker Topology

```javascript
const ZB = require('zeebe-node')

void (async () => {
	const zbc = new ZB.ZBClient('localhost:26500')
	const topology = await zbc.topology()
	console.log(JSON.stringify(topology, null, 2))
})()
```

<a name = "deploy-workflow"></a>

### Deploy a workflow

```javascript
const ZB = require('zeebe-node')
const fs = require('fs')

void (async () => {
	const zbc = new ZB.ZBClient() // localhost:26500 || ZEEBE_GATEWAY_ADDRESS

	const res = await zbc.deployWorkflow('./domain-mutation.bpmn')
	console.log(res)

	// Deploy multiple with an array of filepaths
	await zbc.deployWorkflow(['./wf1.bpmn', './wf2.bpmn'])

	const buffer = fs.readFileSync('./wf3.bpmn')

	// Deploy from an in-memory buffer
	await zbc.deployWorkflow({ definition: buffer, name: 'wf3.bpmn' })
})()
```

## Connection Behaviour

<a name = "client-side-retry"></a>

### Client-side gRPC retry in ZBClient

If a gRPC command method fails in the ZBClient - such as `ZBClient.deployWorkflow` or `ZBClient.topology()`, the underlying gRPC library will throw an exception.

If no workers have been started, this can be fatal to the process if it is not handled by the application logic. This is especially an issue when a worker container starts before the Zeebe gRPC gateway is available to service requests, and can be inconsistent as this is a race condition.

To mitigate against this, the Node client implements some client-side gRPC operation retry logic by default. This can be configured, including disabled, via configuration in the client constructor.

-   Operations retry, but only for [gRPC error codes 8 and 14](https://github.com/grpc/grpc/blob/master/doc/statuscodes.md) - indicating resource exhaustion (8) or transient network failure (14). Resource exhaustion occurs when the broker starts backpressure due to latency because of load. Network failure can be caused by passing in an unresolvable gateway address (`14: DNS Resolution failed`), or by the gateway not being ready yet (`14: UNAVAILABLE: failed to connect to all addresses`).
-   Operations that fail for other reasons, such as deploying an invalid bpmn file or cancelling a workflow that does not exist, do not retry.
-   Retry is enabled by default, and can be disabled by passing { retry: false } to the client constructor.
-   Values for `retry`, `maxRetries` and `maxRetryTimeout` can be configured via the environment variables `ZEEBE_CLIENT_RETRY`, `ZEEBE_CLIENT_MAX_RETRIES` and `ZEEBE_CLIENT_MAX_RETRY_TIMEOUT` respectively.
-   `maxRetries` and `maxRetryTimeout` are also configurable through the constructor options, or through environment variables. By default, if not supplied, the values are:

```TypeScript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient(gatewayAddress, {
    retry: true,
    maxRetries: -1, // infinite retries
    maxRetryTimeout: Duration.seconds.of(5)
})
```

The environment variables are:

```
ZEEBE_CLIENT_MAX_RETRIES
ZEEBE_CLIENT_RETRY
ZEEBE_CLIENT_MAX_RETRY_TIMEOUT
```

Retry is provided by [promise-retry](https://www.npmjs.com/package/promise-retry), and the back-off strategy is simple ^2.

Additionally, the gRPC Client will continually reconnect when in a failed state, such as when the gateway goes away due to pod rescheduling on Kubernetes.

<a name = "eager-connection"></a>

### Eager Connection

The ZBClient eagerly connects to the broker by issuing a topology command in the constructor. This allows you an onReady event to be emitted. You can disable this (for example, for testing without a broker), by either passing `eagerConnection: false` to the client constructor options, or setting the environment variable `ZEEBE_NODE_EAGER_CONNECTION` to `false`.

<a name = "on-ready"></a>

### onReady(), onConnectionError(), and connected

The client has a `connected` property that can be examined to determine if it has a gRPC connection to the gateway.

The client and the worker can take an optional `onReady()` and `onConnectionError()` handler in their constructors, like this:

```TypeScript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient({
	onReady: () => console.log(`Connected!`),
	onConnectionError: () => console.log(`Disconnected!`)
})

const zbWorker = zbc.createWorker({
    taskType: 'demo-service',
	taskHandler: handler,
    onReady: () => console.log(`Worker connected!`),
    onConnectionError: () => console.log(`Worker disconnected!`)
})
```

These handlers are called whenever the gRPC channel is established or lost. As the grpc channel will often "jitter" when it is lost (rapidly emitting READY and ERROR events at the transport layer), there is a `connectionTolerance` property that determines how long the connection must be in a connected or failed state before the handler is called. By default this is 3000ms.

You can specify another value either in the constructor or via an environment variable.

To specify it via an environment variable, set `ZEEBE_CONNECTION_TOLERANCE` to a number of milliseconds.

To set it via the constructor, specify a value for `connectionTolerance` like this:

```TypeScript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient({
	onReady: () => console.log(`Connected!`),
	onConnectionError: () => console.log(`Disconnected!`),
	connectionTolerance: 5000 // milliseconds
})

const zbWorker = zbc.createWorker({
	taskType: 'demo-service',
	taskHandler: handler,
    onReady: () => console.log(`Worker connected!`),
    onConnectionError: () => console.log(`Worker disconnected!`),
    connectionTolerance: Duration.seconds.of(3.5)
})
```

As well as the callback handlers, the client and workers extend the [`EventEmitter`](https://nodejs.org/api/events.html#events_class_eventemitter) class, and you can attach listeners to them for the 'ready' and 'connectionError' events:

```TypeScript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient()

const zbWorker = zbc.createWorker({
	taskType: 'demo-service',
	taskHandler: handler,
    connectionTolerance: Duration.seconds.of(3.5)
})

zbWorker.on('ready', () => console.log(`Worker connected!`))
zbWorker.on('connectionError', () => console.log(`Worker disconnected!`))
```

<a href = "initial-connection-tolerance" >

### Initial Connection Tolerance

Some broker connections can initially emit error messages - for example: when connecting to Camunda Cloud, during TLS negotiation and OAuth authentication, the eager commands used to detect connection status will fail, and the library will report connection errors.

Since this is expected behaviour - a _characteristic of that particular connection_ - the library has a configurable "_initial connection tolerance_". This is a number of milliseconds representing the expected window in which these errors will occur on initial connection.

If the library detects that you are connecting to Camunda Cloud, it sets this window to five seconds (5000 milliseconds). In some environments and under some conditions this may not be sufficient (like connecting to Camunda Cloud from your apartment Wi-fi in South Brisbane, Australia on a rainy day while the microwave link to the next suburb's ADSL exchange is degraded).

You can set an explicit value for this using the environment variable `ZEEBE_INITIAL_CONNECTION_TOLERANCE`, set to a number of milliseconds.

The effect of this setting is to suppress connection errors during this window, and only report them if the connection did not succeed by the end of the window.

## Connecting to a Broker

<a name = "tls"></a>

### TLS

The Node client does not use TLS by default.

Enable a secure connection by setting `useTLS: true`:

```typescript
const { ZBClient } = require('zeebe-node')

const zbc = new ZBClient(tlsSecuredGatewayAddress, {
	useTLS: true,
})
```

Via environment variable:

```bash
ZEEBE_SECURE_CONNECTION=true
```

### Using a Self-signed Certificate

You can use a self-signed SSL certificate with the Zeebe client. You need to provide the root certificates, the private key and the SSL cert chain as Buffers. You can read them in from a file and pass them into the ZBClient constructor:

```
const rootCertsPath = '/path/to/rootCerts'
const privateKeyPath = '/path/to/privateKey'
const certChainPath = '/path/to/certChain'

const zbc = new ZBClient({
    useTLS: true,
    customSSL: {
        rootCerts: rootCertsPath,
        privateKey: privateKeyPath,
        certChain: certChainPath
    }
})

Or you can put the file paths into the environment in the following variables:

```

ZEEBE_CLIENT_SSL_ROOT_CERTS_PATH
ZEEBE_CLIENT_SSL_PRIVATE_KEY_PATH
ZEEBE_CLIENT_SSL_CERT_CHAIN_PATH

# Enable TLS

ZEEBE_SECURE_CONNECTION=true

````

In this case, they will be passed to the constructor automatically.

<a name = "oauth"></a>

### OAuth

In case you need to connect to a secured endpoint with OAuth, you can pass in OAuth credentials. This will enable TLS (unless you explicitly disable it with `useTLS: false`), and handle the OAuth flow to get / renew a JWT:

```typescript
const { ZBClient } = require('zeebe-node')

const zbc = new ZBClient("my-secure-broker.io:443", {
	oAuth: {
		url: "https://your-auth-endpoint/oauth/token",
		audience: "my-secure-broker.io",
		clientId: "myClientId",
		clientSecret: "randomClientSecret",
		cacheOnDisk: true
	}
}
````

The `cacheOnDisk` option will cache the token on disk in `$HOME/.camunda`, which can be useful in development if you are restarting the service frequently, or are running in a serverless environment, like AWS Lambda.

If the cache directory is not writable, the ZBClient constructor will throw an exception. This is considered fatal, as it can lead to denial of service or hefty bills if you think caching is on when it is not.

<a name = "basic-auth"></a>

## Basic Auth

If you put a proxy in front of the broker with basic auth, you can pass in a username and password:

```typescript
const { ZBClient } = require('zeebe-node')

const zbc = new ZBClient("my-broker-with-basic-auth.io:443", {
	basicAuth: {
		username: "user1",
		password: "secret",
	},
	useTLS: true
}
```

Basic Auth will also work without TLS.

<a name = "camunda-cloud"></a>

### Camunda Cloud

[Camunda Cloud](https://camunda.io) is a hosted SaaS instance of Zeebe. The easiest way to connect is to use the [Zero-conf constructor](#zero-conf) with the Client Credentials from the Camunda Cloud console as environment variables.

You can also connect to Camunda Cloud by using the `camundaCloud` configuration option, using the `clusterId`, `clientSecret`, and `clientId` from the Camunda Cloud Console, like this:

```typescript
const { ZBClient } = require('zeebe-node')

const zbc = new ZBClient({
	camundaCloud: {
		clientId,
		clientSecret,
		clusterId,
	},
})
```

That's it! Under the hood, the client lib will construct the OAuth configuration for Camunda Cloud and set the gateway address and port for you.

<a name = "zero-conf"></a>

## Zero-Conf constructor

The ZBClient has a 0-parameter constructor that takes the config from the environment. This is useful for injecting secrets into your app via the environment, and switching between development and production environments with no change to code.

To use the zero-conf constructor, you create the client like this:

```typescript
const { ZBClient } = require('zeebe-node')

const zbc = new ZBClient()
```

With no relevant environment variables set, it will default to localhost on the default port with no TLS.

The following environment variable configurations are possible with the Zero-conf constructor:

Camunda Cloud:

```
ZEEBE_ADDRESS
ZEEBE_AUTHORIZATION_SERVER_URL
ZEEBE_CLIENT_SECRET
ZEEBE_CLIENT_ID
```

Self-hosted or local broker (no TLS or OAuth):

```
ZEEBE_ADDRESS
```

Self-hosted with self-signed SSL certificate:

```
ZEEBE_CLIENT_SSL_ROOT_CERTS_PATH
ZEEBE_CLIENT_SSL_PRIVATE_KEY_PATH
ZEEBE_CLIENT_SSL_CERT_CHAIN_PATH
ZEEBE_SECURE_CONNECTION=true
```

Self-hosted or local broker with OAuth + TLS:

```
ZEEBE_CLIENT_ID
ZEEBE_CLIENT_SECRET
ZEEBE_TOKEN_AUDIENCE
ZEEBE_AUTHORIZATION_SERVER_URL
ZEEBE_ADDRESS
```

Basic Auth:

```
ZEEBE_BASIC_AUTH_PASSWORD
ZEEBE_BASIC_AUTH_USERNAME
```

<a name ="job-workers"></a>

## Job Workers

### Types of Job Workers

There are two different types of job worker provided by the Zeebe Node client:

-   The `ZBWorker` - this worker operates on individual jobs.
-   The `ZBBatchWorker` - this worker batches jobs on the client, to allow you to batch operations that pool resources. (_This worker was introduced in 0.23.0 of the client_).

Much of the information in the following [`ZBWorker` section](#create-zbworker) applies also to the `ZBBatchWorker`. The `ZBBatchWorker` section covers the features that differ from the `ZBWorker`.

<a name = "create-zbworker"></a>

### The `ZBWorker` Job Worker

The `ZBWorker` takes a _job handler function_ that is invoked for each job. It is invoked as soon as the worker retrieves a job from the broker. The worker can retrieve any number of jobs in a response from the broker, and the handler is invoked for each one, independently.

The simplest signature for a worker takes a string task type, and a job handler function.

The job handler receives the job object, a callback that it can use to complete or fail the job, and a reference to the worker itself, which you can use to log using the worker's configured logger (See [Logging](#logging)).

```javascript
const ZB = require('zeebe-node')

const zbc = new ZB.ZBClient()

const zbWorker = zbc.createWorker('demo-service', handler)

function handler(job, complete, worker) {
	worker.log('Task variables', job.variables)

	// Task worker business logic goes here
	const updateToBrokerVariables = {
		updatedProperty: 'newValue',
	}

	complete.success(updateToBrokerVariables)
}
```

Here is an example job:

```javascript

{ key: '578',
  type: 'demo-service',
  jobHeaders:
   { workflowInstanceKey: '574',
     bpmnProcessId: 'test-process',
     workflowDefinitionVersion: 1,
     workflowKey: '3',
     elementId: 'ServiceTask_0xdwuw7',
     elementInstanceKey: '577' },
  customHeaders: '{}',
  worker: 'test-worker',
  retries: 3,
  deadline: '1546915422636',
  variables: { testData: 'something' } }

```

The worker can be configured with options. To do this, you should use the object parameter constructor.

Shown below are the defaults that apply if you don't supply them:

```javascript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient()

const zbWorker = zbc.createWorker({
	taskType: 'demo-service',
    taskHandler: handler,
    // the number of simultaneous tasks this worker can handle
    maxJobsToActivate: 32,
    // the amount of time the broker should allow this worker to complete a task
    timeout: Duration.seconds.of(30),
    // One of 'DEBUG', 'INFO', 'NONE'
    loglevel: 'INFO',
    // Called when the connection to the broker cannot be established, or fails
    onConnectionError: () => zbWorker.log('Disconnected')
    // Called when the connection to the broker is (re-)established
    onReady: () => zbWorker.log('Connected.')
})
```

<a name = "unhandled-exceptions"></a>

#### Unhandled Exceptions in Task Handlers

_Note: this behaviour is for the ZBWorker only. The ZBBatchWorker does not manage this._

When a task handler throws an unhandled exception, the library will fail the job. Zeebe will then retry the job according to the retry settings of the task. Sometimes you want to halt the entire workflow so you can investigate. To have the library cancel the workflow on an unhandled exception, pass in `{failWorkflowOnException: true}` to the `createWorker` call:

```typescript
const { ZBClient } = require('zeebe-node')

const zbc = new ZBClient()

zbc.createWorker('console-log', maybeFaultyHandler, {
	failWorkflowOnException: true,
})
```

<a name = "complete-tasks"></a>

### Completing tasks with success, failure, error, or forwarded

To complete a task, the task worker handler function receives a `complete` parameter. The complete object has `success`, `failure`, and `error` methods.

Call `complete.success()` passing in a optional plain old JavaScript object (POJO) - a key:value map. These are variable:value pairs that will be used to update the workflow state in the broker. They will be merged with existing values. You can set an existing key to `null` or `undefined`, but there is no way to delete a key.

Call `complete.failure()` to fail the task. You must pass in a string message describing the failure. The client library decrements the retry count, and the broker handles the retry logic. If the failure is a hard failure and should cause an incident to be raised in Operate, then pass in `0` for the optional second parameter, `retries`:

```javascript
complete.failure('This is a critical failure and will raise an incident', 0)
```

Call `complete.error()` to trigger a BPMN error throw event. You must pass in a string error code for the error code, and you can pass an optional error message as the second parameter. If no BPMN error catch event exists for the error code, an incident will be raised.

Call `complete.forwarded()` to release worker capacity to handle another job, without completing the job in any way with the Zeebe broker. This method supports the _decoupled job completion_ pattern. In this pattern, the worker forwards the job to another system - a lambda or a RabbitMQ queue. Some other process is ultimately responsible for completing the job.

<a name = "working-with-variables"></a>

## Working with Workflow Variables and Custom Headers

Workflow variables are available in your worker job handler callback as `job.variables`, and any custom headers are available as `job.customHeaders`.

These are read-only JavaScript objects in the Zeebe Node client. However, they are not stored that way in the broker.

Both workflow variables and custom headers are stored in the broker as a dictionary of named strings. That means that the variables and custom headers are JSON.parsed in the Node client when it fetches the job, and any update passed to the `success()` function is JSON.stringified.

If you accidentally pass in a circular JSON structure to `complete()` - like, for example the response object from an HTTP call - it will throw, as this cannot be serialised to a string.

To update a key deep in the object structure of a workflow variable, you can use the [deepmerge utility](https://www.npmjs.com/package/deepmerge):

```TypeScript
const merge = require('deepmerge')
const { ZBClient } = require('zeebe-node')

const zbc = new ZBClient()

zbc.createWorker('some-task', (job, complete) => {
    const { people } = job.variables
    // update bob's age, keeping all his other properties the same
    complete.success(merge(people, { bob: { age: 23 } }))
})
```

When setting custom headers in BPMN tasks, while designing your model, you can put stringified JSON as the value for a custom header, and it will show up in the client as a JavaScript object.

Workflow variables and custom headers are untyped in the Zeebe broker, however the Node client in TypeScript mode provides the option to type them to provide safety. You can type your worker as `any` to turn that off:

```TypeScript
// No type checking - totally dynamic and unchecked
zbc.createWorker<any>({
    taskType: 'yolo-jobs',
    taskHandler: (job, complete, worker) => {
        worker.log(`Look ma - ${job.variables.anything.goes.toUpperCase()}`)
        complete.success({what: job.variables.could.possibly.go.wrong})
    }
})
```

See the section [Writing Strongly-typed Job Workers](#strongly-typed) for more details.

<a name = "fetch-variables"></a>

## Constraining the Variables Fetched by the Worker

Sometimes you only need a few specific workflow variables to service a job. One way you can achieve constraint on the workflow variables received by a worker is by using [input variable mappings](https://docs.zeebe.io/reference/variables.html#inputoutput-variable-mappings) on the task in the model.

You can also use the `fetchVariable` parameter when creating a worker. Pass an array of strings, containing the names of the variables to fetch, to the `fetchVariable` parameter when creating a worker. Here is an example, in JavaScript:

```javascript
zbc.createWorker({
	taskType: 'process-favorite-albums',
	taskHandler: (job, complete, worker) => {
		const { name, albums } = job.variables
		worker.log(`${name} has the following albums: ${albums.join(', ')}`)
		complete.success()
	},
	fetchVariable: ['name', 'albums'],
})
```

If you are using TypeScript, you can supply an interface describing the workflow variables, and parameterize the worker:

```TypeScript
interface Variables {
    name: string
    albums: string[]
}

zbc.createWorker<Variables>({
    taskType: 'process-favorite-albums',
    taskHandler: (job, complete, worker) => {
        const { name, albums = [] } = job.variables
        worker.log(`${name} has the following albums: ${albums?.join?.(', ')}`)
        complete.success()
    },
    fetchVariable: ['name', 'albums'],
})
```

This parameterization does two things:

-   It informs the worker about the expected types of the variables. For example, if `albums` is a string, calling `join` on it will fail at runtime. Providing the type allows the compiler to reason about the valid methods that can be applied to the variables.
-   It allows the type-checker to pick up spelling errors in the strings in `fetchVariable`, by comparing them with the Variables typing.

Note, that this does not protect you against run-time exceptions where your typings are incorrect, or the payload simply does not match the definition that you provided.

See the section [ Writing Strongly-typed Job Workers ](#strongly-typed) for more details on run-time safety.

You can turn off the type-safety by typing the worker as `any`:

```TypeScript
zbc.createWorker<any>({
    taskType: 'process-favorite-albums',
    taskHandler: (job, complete, worker) => {
        const { name, albums = [] } = job.variables
        // TS 3.7 safe access to .join _and_ safe call, to prevent run-time exceptions
        worker.log(`${name} has the following albums: ${albums?.join?.(', ')}`)
        complete.success()
    },
    fetchVariable: ['name', 'albums'],
})
```

<a name = "decoupled-complete"></a>

## The "Decoupled Job Completion" pattern

The _Decoupled Job Completion_ pattern uses a Zeebe Job Worker to activate jobs from the broker, and some other asynchronous (remote) system to do the work.

You might activate jobs and then send them to a RabbitMQ queue, or to an AWS lambda. In this case, there may be no outcome about the job that this worker can report back to the broker about success or failure. That will be the responsibility of another part of your distributed system.

The first thing you should do is ensure that you activate the job with sufficient time for the complete execution of your system. Your worker will not be completing the job, but it informs the broker how long the expected loop will take to close.

Next, call `complete.forwarded()` in your job worker handler. This has no side-effect with the broker - so nothing is communicated to Zeebe. The job is still out there with your worker as far as Zeebe is concerned. What this call does is release worker capacity to request more jobs.

If you are using the Zeebe Node library in the remote system, or if the remote system eventually reports back to you (perhaps over a different RabbitMQ queue), you can use the ZBClient methods `completeJob()`, `failJob()`, and `throwError()` to report the outcome back to the broker.

You need at least the `job.key`, to be able to correlate the result back to Zeebe. Presumably you also want the information from the remote system about the outcome, and any updated variables.

Here is an example:

-   You have a COBOL system that runs a database.
-   Somebody wrote an adapter for this COBOL database. In executes commands over SSH.
-   The adapter is accessible via a RabbitMQ "request" queue, which takes a command and a correlation id, so that its response can be correlated to this request.
-   The adapter sends back the COBOL database system response on a RabbitMQ "response" queue, with the correlation id.
-   It typically takes 15 seconds for the round-trip through RabbitMQ to the COBOL database and back.

You want to put this system into a Zeebe-orchestrated BPMN model as a task.

Rather than injecting a RabbitMQ listener into the job handler, you can "_fire and forget_" the request using the decoupled job completion pattern.

Here is how you do it:

-   Your worker gets the job from Zeebe.
-   Your worker makes the command and sends it down the RabbitMQ "request" queue, with the `job.jobKey` as the correlation id.
-   Your worker calls `complete.forwarded()`

Here is what that looks like in code:

```TypeScript
import { RabbitMQSender } from './lib/my-awesome-rabbitmq-api'
import { ZBClient, Duration } from 'zeebe-node'

const zbc = new ZBClient()

const cobolWorker = zbc.createWorker({
    taskType: 'cobol-insert',
    timeout: Duration.seconds.of(20), // allow 5s over the expected 15s
    taskHandler: (job, complete) => {
    const { jobKey, variables } = job
    const request = {
        correlationId: jobKey,
        command: `INSERT ${variables.customer} INTO CUSTOMERS`
    }
    RabbitMQSender.send({
        channel: 'COBOL_REQ',
        request
    })
    // Call forwarded() to release worker capacity
    complete.forwarded()
})
```

Now for the response part:

-   Another part of your system listens to the RabbitMQ response queue.
-   It gets a response back from the COBOL adapter.
-   It examines the response, then sends the appropriate outcome to Zeebe, using the jobKey that has been attached as the correlationId

```TypeScript
import { RabbitMQListener } from './lib/my-awesome-rabbitmq-api'
import { ZBClient } from 'zeebe-node'

const zbc = new ZBClient()

const RabbitMQListener.listen({
    channel: 'COBOL_RES',
    handler: message => {
        const { outcome, correlationId } = message
        if (outcome.SUCCESS) {
            zbc.completeJob({
                jobKey: correlationId,
            })
        }
        if (outcome.ERROR) {
            zbc.throwError({
                jobKey: correlationId,
                errorCode: "5",
                errorMessage: "The COBOL Database reported an error. Boo!"
            })
        }
    })
}
```

See also the section "[Publish a Message](#publish-message)", for a pattern that you can use when it is not possible to attach the job key to the round trip data response.

<a name = "zbbatchworker"></a>

## The `ZBBatchWorker` Job Worker

The `ZBBatchWorker` Job Worker batches jobs before calling the job handler. Its fundamental differences from the ZBWorker are:

-   Its job handler receives an _array_ of one or more jobs.
-   The jobs have `success`, `failure`, `error`, and `forwarded` methods _attached_ to them.
-   The handler is not invoked immediately, but rather when enough jobs are batched, or a job in the batch is at risk of being timed out by the Zeebe broker.

You can use the batch worker if you have tasks that _benefit from processing together_, but are _not related in the BPMN model_.

An example would be a high volume of jobs that require calls to an external system, where you have to pay per call to that system. In that case, you may want to batch up jobs, make one call to the external system, then update all the jobs and send them on their way.

The batch worker works on a _first-of_ batch size _or_ batch timeout basis.

You must configure both `jobBatchMinSize` and `jobBatchMaxTime`. Whichever condition is met first will trigger the processing of the jobs:

-   Enough jobs are available to the worker to satisfy the minimum job batch size;
-   The batch has been building for the maximum amount of time - "_we're doing this now, before the earliest jobs in the batch time out on the broker_".

You should be sure to specify a `timeout` for your worker that is `jobBatchMaxTime` _plus_ the expected latency of the external call _plus_ your processing time and network latency, to avoid the broker timing your batch worker's lock and making the jobs available to another worker. That would defeat the whole purpose.

Here is an example of using the `ZBBatchWorker`:

```TypeScript
import { API } from './lib/my-awesome-external-api'
import { ZBClient, BatchedJob, Duration } from 'zeebe-node'

const zbc = new ZBClient()

// Helper function to find a job by its key
const findJobByKey = jobs => key => jobs.filter(job => job.jobKey === id)?.[0] ?? []

const handler = async (jobs: BatchedJob[], worker: ZBBatchWorker) => {
    worker.log("Let's do this!")
    const {jobKey, variables} = job
    // Construct some hypothetical payload with correlation ids and requests
    const req = jobs.map(job => ({id: jobKey, data: variables.request}))
    // An uncaught exception will not be managed by the library
    try {
        // Our API wrapper turns that into a request, and returns
        // an array of results with ids
        const outcomes = await API.post(req)
        // Construct a find function for these jobs
        const getJob = findJobByKey(jobs)
        // Iterate over the results and call the succeed method on the corresponding job,
        // passing in the correlated outcome of the API call
        outcomes.forEach(res => getJob(res.id)?.success(res.data))
    } catch (e) {
        jobs.forEach(job => job.failure(e.message))
    }
}

const batchWorker = zbc.createBatchWorker({
    taskType: 'get-data-from-external-api',
    taskHandler: handler,
    jobBatchMinSize: 10, // at least 10 at a time
    jobBatchMaxTime: 60, // or every 60 seconds, whichever comes first
    timeout: Duration.seconds.of(80) // 80 second timeout means we have 20 seconds to process at least
})
```

See [this blog post](http://joshwulf.com/blog/2020/03/zb-batch-worker/) for some more details on the implementation.

<a name = "long-polling"></a>

### Long polling

With Zeebe 0.21 onward, long polling is supported for clients, and is used by default. Rather than polling continuously for work and getting nothing back, a client can poll once and leave the request open until work appears. This reduces network traffic and CPU utilization in the server. Every JobActivation Request is appended to the event log, so continuous polling can significantly impact broker performance, especially when an exporter is loaded (see [here](https://github.com/zeebe-io/zeebe-client-node-js/issues/64#issuecomment-520233275)).

The default long polling period is 30s.

To use a different long polling period, pass in a long poll timeout in milliseconds to the client. All workers created with that client will use it. Alternatively, set a period per-worker.

Long polling for workers is configured in the ZBClient like this:

```typescript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient('serverAddress', {
	longPoll: Duration.minutes.of(10), // Ten minutes - inherited by workers
})

const longPollingWorker = zbc.createWorker({
	taskType: 'task-type',
	taskHandler: handler,
	longPoll: Duration.minutes.of(2), // override client, poll 2m
})
```

## Client Commands

<a name = "start-workflow"></a>

### Start a Workflow Instance

```javascript
const ZB = require('zeebe-node')

;(async () => {
	const zbc = new ZB.ZBClient('localhost:26500')
	const result = await zbc.createWorkflowInstance('test-process', {
		testData: 'something',
	})
	console.log(result)
})()
```

Example output:

```javascript

{ workflowKey: '3',
  bpmnProcessId: 'test-process',
  version: 1,
  workflowInstanceKey: '569' }

```

<a name = "start-specific-version"></a>

### Start a Workflow Instance of a specific version of a Workflow definition

From version 0.22 of the client onward:

```javascript
const ZB = require('zeebe-node')

;(async () => {
	const zbc = new ZB.ZBClient('localhost:26500')
	const result = await zbc.createWorkflowInstance({
		bpmnProcessId: 'test-process',
		variables: {
			testData: 'something',
		},
		version: 5,
	})
	console.log(result)
})()
```

<a name = "start-await"></a>

### Start a Workflow Instance and await the Workflow Outcome

From version 0.22 of the broker and client, you can await the outcome of a workflow end-to-end execution:

```typescript
async function getOutcome() {
	const result = await zbc.createWorkflowInstanceWithResult(processId, {
		sourceValue: 5,
	})
	return result
}
```

Be aware that by default, **this will throw an exception if the workflow takes longer than 15 seconds to complete**.

To override the gateway's default timeout for a workflow that needs more time to complete:

```typescript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient()

const result = await zbc.createWorkflowInstanceWithResult({
	bpmnProcessId: processId,
	variables: {
		sourceValue: 5,
		otherValue: 'rome',
	},
	requestTimeout: Duration.seconds.of(25),
	// also works supplying a number of milliseconds
	// requestTimeout: 25000
})
```

<a name = "publish-message"></a>

### Publish a Message

You can publish a message to the Zeebe broker that will be correlated with a running workflow instance:

```javascript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZBClient()

zbc.publishMessage({
	correlationKey: 'value-to-correlate-with-workflow-variable',
	messageId: uuid.v4(),
	name: 'message-name',
	variables: { valueToAddToWorkflowVariables: 'here', status: 'PROCESSED' },
	timeToLive: Duration.seconds.of(10), // seconds
})
```

When would you do this? Well, the sky is not even the limit when it comes to thinking creatively about building a system with Zeebe - _and_ here's one concrete example to get you thinking:

Recall the example of the _remote COBOL database_ in the section "[The "Decoupled Job Completion" pattern](#decoupled-complete)". We're writing code to allow that system to be participate in a BPMN-modelling workflow orchestrated by Zeebe.

But what happens if the adapter for that system has been written in such a way that there is no opportunity to attach metadata to it? In that case we have no opportunity to attach a job key. Maybe you send the fixed data for the command, and you have to correlate the response based on those fields.

Another example: think of a system that emits events, and has no knowledge of a running workflow. An example from one system that I orchestrate with Zeebe is Minecraft. A logged-in user in the game performs some action, and code in the game emits an event. I can catch that event in my Node-based application, but I have no knowledge of which running workflow to target - _and_ the event was not generated from a BPMN task providing a worker with the complete context of a workflow.

In these two cases, I can publish a message to Zeebe, and let the broker figure out which workflows are:

-   Sitting at an intermediate message catch event waiting for this message; or
-   In a sub-process that has a boundary event that will be triggered by this message; or
-   Would be started by a message start event, on receiving this message.

The Zeebe broker correlates a message to a running workflow instance _not on the job key_ - but on _the value of one of the workflow variables_ (for intermediate message events) and _the message name_ (for all message events, including start messages).

So the response from your COBOL database system, sans job key, is sent back to Zeebe from the RabbitMQListener not via `completeJob()`, but with `publishMessage()`, and the value of the payload is used to figure out which workflow it is for.

In the case of the Minecraft event, a message is published to Zeebe with the Minecraft username, and that is used by Zeebe to determine which workflows are running for that user and are interested in that event.

See the article "[Zeebe Message Correlation](https://zeebe.io/blog/2019/08/zeebe-message-correlation/)" for a complete example with code.

<a name="publish-start-message"></a>

### Publish a Start Message

You can also publish a message targeting a [Message Start Event](https://github.com/zeebe-io/zeebe/issues/1858).
In this case, the correlation key is optional, and all Message Start events that match the `name` property will receive the message.

You can use the `publishStartMessage()` method to publish a message with no correlation key (it will be set to a random uuid in the background):

```javascript
const { ZBClient, Duration } = require('zeebe-node')

const zbc = new ZB.ZBClient('localhost:26500')
zbc.publishStartMessage({
	messageId: uuid.v4(),
	name: 'message-name',
	variables: { initialWorkflowVariable: 'here' },
	timeToLive: Duration.seconds.of(10), // seconds
})
```

Both normal messages and start messages can be published idempotently by setting both the `messageId` and the `correlationKey`. They will only ever be correlated once. See: [A message can be published idempotent](https://github.com/zeebe-io/zeebe/issues/1012).

<a name="activate-jobs"></a>

### Activate Jobs

If you have some use case that doesn't fit the existing workers, you can write your own custom workers using the `ZBClient.activateJobs()` method. It takes an `ActivateJobsRequest` object, and returns a stream for that call.

Attach a listener to the stream's 'data' event, and it will be called with an `ActivateJobsResponse` object if there are jobs to work on.

To complete these jobs, use the `ZBClient` methods `completeJob()`, `failJob()`, and `throwError()`.

For more details, read the source code of the library, particularly the `ZBWorkerBase` class. This is an advanced use case, and the existing code in the library is the best documentation.

## Other Concerns

<a name = "graceful-shutdown"></a>

### Graceful Shutdown

To drain workers, call the `close()` method of the ZBClient. This causes all workers using that client to stop polling for jobs, and returns a Promise that resolves when all active jobs have either finished or timed out.

```javascript
console.log('Closing client...')
zbc.close().then(() => console.log('All workers closed'))
```

<a name = "logging"></a>

## Logging

Control the log output for the client library by setting the ZBClient log level. Valid log levels are `NONE` (supress all logging), `ERROR` (log only exceptions), `INFO` (general logging), or `DEBUG` (verbose logging). You can set this in the client constructor:

```typescript
const zbc = new ZBClient('localhost', { loglevel: 'DEBUG' })
```

And also via the environment:

```bash
ZEEBE_NODE_LOG_LEVEL='ERROR' node start.js
```

By default the library uses `console.info` and `console.error` for logging. You can also pass in a custom logger, such as [pino](https://github.com/pinojs/pino):

```typescript
const logger = require('pino')()
const zbc = new ZBClient({ stdout: logger })
```

From version v0.23.0-alpha.1, the library logs human-readable logs by default, using the `ZBSimpleLogger`. If you want structured logs as stringified JSON, pass in `ZBJSONLogger` to the constructor `stdout` option, like this:

```typescript
const { ZBJsonLogger, ZBClient } = require('zeebe-node')
const zbc = new ZBClient({ stdout: ZBJsonLogger })
```

You can also control this via environment variables:

```bash
export ZEEBE_NODE_LOG_TYPE=SIMPLE  # Simple Logger (default)
export ZEEBE_NODE_LOG_TYPE=JSON  # JSON Logger
```

<a name = "generate-constants"></a>

### Generating TypeScript constants for BPMN Models

Message names and Task Types are untyped magic strings. You can generate type information to avoid some classes of errors.

#### 0.22.0-alpha.5 and above

Install the package globally:

```
npm i -g zeebe-node
```

Now you have the command `zeebe-node <filename>` that parses a BPMN file and emits type definitions.

#### All versions

The `BpmnParser` class provides a static method `generateConstantsForBpmnFiles()`.
This method takes a filepath and returns TypeScript definitions that you can use to avoid typos in your code, and to reason about the completeness of your task worker coverage.

```javascript
const ZB = require('zeebe-node')
;(async () => {
	console.log(await ZB.BpmnParser.generateConstantsForBpmnFiles(workflowFile))
})()
```

This will produce output similar to:

```typescript
// Autogenerated constants for msg-start.bpmn

export enum TaskType = {
    CONSOLE_LOG = "console-log"
};

export enum MessageName = {
    MSG_EMIT_FRAME = "MSG-EMIT_FRAME",
    MSG_START_JOB = "MSG-START_JOB"
};
```

<a name = "generate-code"></a>

## Generating code from a BPM Model file

You can scaffold your worker code from a BPMN file with the `zeebe-node` command. To use this command, install the package globally with:

```bash
npm i -g zeebe-node
```

Pass in the path to the BPMN file, and it will output a file to implement it:

```bash
zeebe-node my-model.bpmn
```

<a name = "strongly-typed"></a>

### Writing Strongly-typed Job Workers

You can provide interfaces to get design-time type safety and intellisense on the workflow variables passed in the a worker job handler, the custom headers that it will receive, and the variables that it will pass back to Zeebe in the `complete.success` call:

```TypeScript
interface InputVariables {
    name: string,
    age: number,
    preferences: {
        beverage: 'Coffee' | 'Tea' | 'Beer' | 'Water',
        color: string
    }
}

interface OutputVariables {
    suggestedGift: string
}

interface CustomHeaders {
    occasion: 'Birthday' | 'Christmas' | 'Hannukah' | 'Diwali'
}

const giftSuggester = zbc.createWorker<
    InputVariables,
    CustomHeaders,
    OutputVariables>
    ('get-gift-suggestion', (job, complete) => {
        const suggestedGift = `${job.customHeaders.occasion} ${job.variables.preferences.beverage}`
        complete.success({ suggestedGift })
})
```

If you decouple the declaration of the job handler from the `createWorker` call, you will need to explicitly specify its type, like this:

```TypeScript
import { ZBWorkerTaskHandler } from 'zeebe-node'

function getGiftSuggestion(job, complete): ZBWorkerTaskHandler<InputVariables, CustomHeaders, OutputVariables> {
    const suggestedGift = `${job.customHeaders.occasion} ${job.variables.preferences.beverage}`
    complete.success({ suggestedGift })
}

const giftSuggester = zbc.createWorker('get-gift-suggestion', getGiftSuggestion)

```

<a name = "run-time-safety"></a>

## Run-time Type Safety

The parameterization of the client and workers helps to catch errors in code, and if your interface definitions are good, can go a long way to making sure that your workers and client emit the correct payloads and have a strong expectation about what they will receive, but it does not give you any _run-time safety_.

Your type definition may be incorrect, or the variables or custom headers may simply not be there at run-time, as there is no type checking in the broker, and other factors are involved, such as tasks with input and output mappings, and data added to the workflow variables by REST calls and other workers.

You should consider:

-   Writing interface definitions for your payloads to get design-time assist for protection against spelling errors as you demarshal and update variables.
-   Testing for the existence of variables and properties on payloads, and writing defensive pathways to deal with missing properties. If you mark _everything_ as optional in your interfaces, the type-checker will force you to write that code.
-   Surfacing code exceptions operationally to detect and diagnose mismatched expectations.
-   If you want to validate inputs and outputs to your system at runtime, you can use [io-ts](https://github.com/gcanti/io-ts). Once data goes into that, it either exits through an exception handler, or is guaranteed to have the shape of the defined codec at run-time.

As with everything, it is a balancing act / trade-off between correctness, safety, and speed. You do not want to lock everything down while you are still exploring.

I recommend the following scale, to match the maturity of your system:

-   Start with `<any>` typing for the workers; then
-   Develop interfaces to describe the DTOs represented in your workflow variables;
-   Use optional types on those interfaces to check your defensive programming structures;
-   Lock down the run-time behaviour with io-ts as the boundary validator.

You may choose to start with the DTOs. Anyway, there are options.

<a name = "developing"></a>

## Developing Zeebe Node

The source is written in TypeScript in `src`, and compiled to ES6 in the `dist` directory.

To build:

```bash
npm run build
```

To start a watcher to build the source and API docs while you are developing:

```bash
npm run dev
```

<a name = "tests"></a>

### Tests

Tests are written in Jest, and live in the `src/__tests__` directory. To run the unit tests:

```bash
npm t
```

Integration tests are in the `src/__tests__/integration` directory.

They require a Zeebe broker to run. You can run them using the [Circle CI CLI](https://circleci.com/docs/2.0/local-cli/):

```bash
circleci local execute -c .circleci/config.yml --job test
```

Or you can start a dockerised broker:

```bash
cd docker
docker-compose up
```

And then run them manually:

```bash
npm run test:integration
```

For the failure test, you need to run Operate ([docker-compose config](https://github.com/zeebe-io/zeebe-docker-compose/blob/master/operate/docker-compose.yml)) and manually verify that an incident has been raised at [http://localhost:8080](http://localhost:8080).

<a name = "writing-tests"></a>

### Writing Tests

Zeebe is inherently stateful, so integration tests need to be carefully isolated so that workers from one test do not service tasks in another test. Jest runs tests in a random order, so intermittent failures are the outcome of tests that mutate shared state.

The tests use a templating function to replace the process id, task types and message names in the bpmn model to produce distinct, isolated namespaces for each test and each test run.

<a name = "contributors"></a>

## Contributors

| Name                                                         |
| ------------------------------------------------------------ |
| **[Josh Wulf](https://github.com/jwulf)**                    |
| **[Colin Raddatz](https://github.com/ColRad)**               |
| **[Jarred Filmer](https://github.com/BrighTide)**            |
| **[Timothy Colbert](https://github.com/s3than)**             |
| **[Olivier Albertini](https://github.com/OlivierAlbertini)** |
| **[Patrick Dehn](https://github.com/pedesen)**               |
