# graphql-mqtt-subscriptions

This package implements the PubSubEngine Interface from the graphql-subscriptions package. 
It allows you to connect your subscriptions manager to an MQTT enabled Pub Sub broker to support 
horizontally scalable subscriptions setup.
This package is an adapted version of my [graphql-redis-subscriptions](https://github.com/davidyaha/graphql-redis-subscriptions) package.
   
   
## Basic Usage

```javascript
import { MQTTPubSub } from 'graphql-mqtt-subscriptions';
const pubsub = new MQTTPubSub(); // connecting to mqtt://localhost on default
const subscriptionManager = new SubscriptionManager({
  schema,
  pubsub,
  setupFunctions: {},
});
```

## Using Trigger Transform

As the [graphql-redis-subscriptions](https://github.com/davidyaha/graphql-redis-subscriptions) package, this package support
a trigger transform function. This trigger transform allows you to use the `channelOptions` object provided to the `SubscriptionManager`
instance, and return trigger string which is more detailed then the regular trigger. 

First I create a simple and generic trigger transform 
```javascript
const triggerTransform = (trigger, {path}) => [trigger, ...path].join('.');
```
> Note that I expect a `path` field to be passed to the `channelOptions` but you can do whatever you want.

Next, I'll pass the `triggerTransform` to the `MQTTPubSub` constructor.
```javascript
const pubsub = new MQTTPubSub({
  triggerTransform,
});
```
Lastly, I provide a setupFunction for `commentsAdded` subscription field.
It specifies one trigger called `comments.added` and it is called with the `channelOptions` object that holds `repoName` path fragment.
```javascript
const subscriptionManager = new SubscriptionManager({
  schema,
  setupFunctions: {
    commentsAdded: (options, {repoName}) => ({
      'comments/added': {
        channelOptions: { path: [repoName] },
      },
    }),
  },
  pubsub,
});
```
> Note that here is where I satisfy my `triggerTransform` dependency on the `path` field.

When I call `subscribe` like this:
```javascript
const query = `
  subscription X($repoName: String!) {
    commentsAdded(repoName: $repoName)
  }
`;
const variables = {repoName: 'graphql-redis-subscriptions'};
subscriptionManager.subscribe({ query, operationName: 'X', variables, callback });
```

The subscription string that Redis will receive will be `comments.added.graphql-redis-subscriptions`.
This subscription string is much more specific and means the the filtering required for this type of subscription is not needed anymore.
This is one step towards lifting the load off of the graphql api server regarding subscriptions.

## Passing your own client object

The basic usage is great for development and you will be able to connect to any mqtt enabled server running on your system seamlessly.
But for any production usage you should probably pass in your own configured client object;
 
```javascript
import { connect } from 'mqtt';
import { MQTTPubSub } from 'graphql-mqtt-subscriptions';

const client = connect('mqtt://test.mosquitto.org', {
  reconnectPeriod: 1000,
});

const pubsub = new MQTTPubSub({
  client,
});
```

You can learn more on the mqtt options object [here](https://github.com/mqttjs/MQTT.js#client).

## Changing QoS for publications or subscriptions

As specified [here](https://github.com/mqttjs/MQTT.js#publish), the MQTT.js publish and subscribe functions takes an 
option object. This object could be defined per trigger with `publishOptions` and `subscribeOptions` resolvers.

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

## Get notified on the actual QoS assigned for subscription

MQTT allows the broker to assign different QoS level than the one requested by the client. 
In order to know what QoS was defined for your subscription you can pass in a callback called `onMQTTSubscribe`

```javascript
const onMQTTSubscribe = (subId, granted) => {
  console.log(`Subscription with id ${subId} was given QoS of ${granted.qos}`);
}

const pubsub = new MQTTPubSub({onMQTTSubscribe});
```

## Change encoding used to encode and decode messages

Supported encodings available [here](https://nodejs.org/api/buffer.html#buffer_buffers_and_character_encodings) 

```javascript
const pubsub = new MQTTPubSub({
  parseMessageWithEncoding: 'utf16le',
});
```
