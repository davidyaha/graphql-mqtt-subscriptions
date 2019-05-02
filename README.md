# graphql-mqtt-subscriptions

This package implements the AsyncIterator Interface and PubSubEngine Interface from the [graphql-subscriptions](https://github.com/apollographql/graphql-subscriptions) package. 
It allows you to connect your subscriptions manager to an MQTT enabled Pub Sub broker to support 
horizontally scalable subscriptions setup.
This package is an adapted version of my [graphql-redis-subscriptions](https://github.com/davidyaha/graphql-redis-subscriptions) package.

## Installation

```
npm install graphql-mqtt-subscriptions
```

## Using the AsyncIterator Interface

Define your GraphQL schema with a `Subscription` type.

```graphql
schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}

type Subscription {
    somethingChanged: Result
}

type Result {
    id: String
}
```

Now, create a `MQTTPubSub` instance.

```javascript
import { MQTTPubSub } from 'graphql-mqtt-subscriptions';
const pubsub = new MQTTPubSub(); // connecting to mqtt://localhost by default
```

Now, implement the Subscriptions type resolver, using `pubsub.asyncIterator` to map the event you need.

```javascript
const SOMETHING_CHANGED_TOPIC = 'something_changed';

export const resolvers = {
  Subscription: {
    somethingChanged: {
      subscribe: () => pubsub.asyncIterator(SOMETHING_CHANGED_TOPIC)
    }
  }
}
```

> Subscriptions resolvers are not a function, but an object with `subscribe` method, that returns `AsyncIterable`.

The `AsyncIterator` method will tell the MQTT client to listen for messages from the MQTT broker on the topic provided, and wraps that listener in an `AsyncIterator` object. 

When messages are received from the topic, those messages can be returned back to connected clients.

`pubsub.publish` can be used to send messages to a given topic.

```js
pubsub.publish(SOMETHING_CHANGED_TOPIC, { somethingChanged: { id: "123" }});
```

## Dynamically Create a Topic Based on Subscription Args Passed on the Query:

```javascript
export const resolvers = {
  Subscription: {
    somethingChanged: {
      subscribe: (_, args) => pubsub.asyncIterator(`${SOMETHING_CHANGED_TOPIC}.${args.relevantId}`),
    },
  },
}
```

## Using Arguments and Payload to Filter Events

```javascript
import { withFilter } from 'graphql-subscriptions';

export const resolvers = {
  Subscription: {
    somethingChanged: {
      subscribe: withFilter(
        (_, args) => pubsub.asyncIterator(`${SOMETHING_CHANGED_TOPIC}.${args.relevantId}`),
        (payload, variables) => payload.somethingChanged.id === variables.relevantId,
      ),
    },
  },
}
```

## Passing your own client object

The basic usage is great for development and you will be able to connect to any mqtt enabled server running on your system seamlessly.
For production usage, it is recommended you pass your own MQTT client.
 
```javascript
import { connect } from 'mqtt';
import { MQTTPubSub } from 'graphql-mqtt-subscriptions';

const client = connect('mqtt://test.mosquitto.org', {
  reconnectPeriod: 1000,
});

const pubsub = new MQTTPubSub({
  client
});
```

You can learn more on the mqtt options object [here](https://github.com/mqttjs/MQTT.js#client).

## Changing QoS for publications or subscriptions

As specified [here](https://github.com/mqttjs/MQTT.js#publish), the MQTT.js publish and subscribe functions takes an 
options object. This object can be defined per trigger with `publishOptions` and `subscribeOptions` resolvers.

```javascript
const triggerToQoSMap = {
  'comments.added': 1,
  'comments.updated': 2,
};

const pubsub = new MQTTPubSub({
  publishOptions: trigger => Promise.resolve({ qos: triggerToQoSMap[trigger] }),
  
  subscribeOptions: (trigger, channelOptions) => Promise.resolve({ 
    qos: Math.max(triggerToQoSMap[trigger], channelOptions.maxQoS), 
  }),
});
```

## Get Notified of the Actual QoS Assigned for a Subscription

MQTT allows the broker to assign different QoS levels than the one requested by the client. 
In order to know what QoS was given to your subscription, you can pass in a callback called `onMQTTSubscribe`

```javascript
const onMQTTSubscribe = (subId, granted) => {
  console.log(`Subscription with id ${subId} was given QoS of ${granted.qos}`);
}

const pubsub = new MQTTPubSub({onMQTTSubscribe});
```

## Change Encoding Used to Encode and Decode Messages

Supported encodings available [here](https://nodejs.org/api/buffer.html#buffer_buffers_and_character_encodings) 

```javascript
const pubsub = new MQTTPubSub({
  parseMessageWithEncoding: 'utf16le',
});
```


## Basic Usage with SubscriptionManager (Deprecated)

```javascript
import { MQTTPubSub } from 'graphql-mqtt-subscriptions';
const pubsub = new MQTTPubSub(); // connecting to mqtt://localhost on default
const subscriptionManager = new SubscriptionManager({
  schema,
  pubsub,
  setupFunctions: {},
});
```

## Using Trigger Transform (Deprecated)

Similar to the [graphql-redis-subscriptions](https://github.com/davidyaha/graphql-redis-subscriptions) package, this package supports
a trigger transform function. This trigger transform allows you to use the `channelOptions` object provided to the `SubscriptionManager`
instance, and return a trigger string which is more detailed then the regular trigger. 

Here is an example of a generic trigger transform.

```javascript
const triggerTransform = (trigger, { path }) => [trigger, ...path].join('.');
```

> Note that a `path` field to be passed to the `channelOptions` but you can do whatever you want.

Next, pass the `triggerTransform` to the `MQTTPubSub` constructor.

```javascript
const pubsub = new MQTTPubSub({
  triggerTransform,
});
```

Lastly, a setupFunction is provided for the `commentsAdded` subscription field.
It specifies one trigger called `comments.added` and it is called with the `channelOptions` object that holds `repoName` path fragment.
```javascript
const subscriptionManager = new SubscriptionManager({
  schema,
  setupFunctions: {
    commentsAdded: (options, { repoName }) => ({
      'comments/added': {
        channelOptions: { path: [repoName] },
      },
    }),
  },
  pubsub,
});
```
> Note that the `triggerTransform` dependency on the `path` field is satisfied here.

When `subscribe` is called like this:

```javascript
const query = `
  subscription X($repoName: String!) {
    commentsAdded(repoName: $repoName)
  }
`;
const variables = {repoName: 'graphql-mqtt-subscriptions'};
subscriptionManager.subscribe({ query, operationName: 'X', variables, callback });
```

The subscription string that MQTT will receive will be `comments.added.graphql-mqtt-subscriptions`.
This subscription string is much more specific and means the the filtering required for this type of subscription is not needed anymore.
This is one step towards lifting the load off of the graphql api server regarding subscriptions.