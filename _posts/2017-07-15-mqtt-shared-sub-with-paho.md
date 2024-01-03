---
layout:     post
title:      "MQTT shared subscriptions with Paho"
date:       2017-07-15 11:43:58 +0530
comments:   true
---
Shared subscriptions are great way to [load balance](http://www.hivemq.com/blog/mqtt-client-load-balancing-with-shared-subscriptions/) the client listeners for MQTT's subscribers. However the [Erlang's MQTT broker](http://emqtt.io/docs/v2/advanced.html) supports shared subscriptions; since it is not outlined in MQTT 3.1.1 specification, Paho client doesn't comply with shared subscription format. This blog deals with Paho's limitation.

Paho is famous client for MQTT, we used that to deploy our chat application to connect with EMQ broker. But Paho's client doesn't work with shared topic subscriptions. [https://github.com/eclipse/paho.mqtt.java/issues/367](https://github.com/eclipse/paho.mqtt.java/issues/367) . There are many MQTT brokers which supports shared subscription of their flavour. MQTTv5 would include shared subscriptions and Paho client would support them in Java client v5. But the 3.1.1 client we needed to add custom router.

EMQ's shared topic looks something like this:
```java
    String sharedTopic = "$shared/group1/test/topic"; //publishers would publish on "test/topic"
```

## Shared subscriptions

Shared subscriptions are useful in load balance.
```java
package com.example.testing;

import org.apache.commons.io.IOUtils;
import org.eclipse.paho.client.mqttv3.*;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.junit.Test;

import java.util.concurrent.TimeUnit;

/**
 * Date 08/05/17
 * Time 11:31 AM
 *
 * @author yogin
 */
public class MqttSharedSubsTest {
    private String serverUrl = "ws://localhost:8083";

    @Test
    public void testServerA() throws Exception {
        MqttClient client = getClient(serverUrl, "s1");
        connectClient(client);

        client.subscribe("$queue//test/+/topic", new IMqttMessageListener() { //This isn't called by paho, even though broker delivers the message, this is not called because topic is not matched.
                    @Override
                    public void messageArrived(String topic, MqttMessage message) throws Exception {
                        System.out.println("message arrived on`` serverA topic: " + topic + " message: " + IOUtils.toString(message.getPayload()));
                    }
                });
        System.out.println("serverA connected");
        Thread.sleep(TimeUnit.HOURS.toMillis(1));
    }

    @Test
    public void testServerB() throws Exception {
        MqttClient client = getClient(serverUrl, "s2");
        connectClient(client);

        client.subscribe("$queue//test/+/topic", new IMqttMessageListener() { //This isn't called by paho, even though broker delivers the message, this is not called because topic is not matched.
                    @Override
                    public void messageArrived(String topic, MqttMessage message) throws Exception {
                        System.out.println("message arrived on`` serverB topic: " + topic + " message: " + IOUtils.toString(message.getPayload()));
                    }
                });
        System.out.println("serverB connected");
        Thread.sleep(TimeUnit.HOURS.toMillis(1));
    }

    @Test
    public void testClientPublish() throws Exception {
        MqttClient client = getClient(serverUrl, "abc");
        connectClient(client);
        System.out.println("publisbing messages");
        for (int i = 0; i < 100; i++) {
            client.publish("/test/1234/topic", ("message no: " + i).getBytes(), 0, false);
        }
        System.out.println("Done publishing messages");
    }

    private MqttClient getClient(String mqttServerUrl) {
        return getClient(mqttServerUrl, "dummy-test-configuration-client");
    }

    private MqttClient getClient(String mqttServerUrl, String clientId) {
        try {
            return new MqttClient(mqttServerUrl, "dummy-test-" + clientId, new MemoryPersistence());
        } catch (MqttException e) {
            throw new RuntimeException("Could not initialize mqtt client", e);
        }
    }

    private void connectClient(MqttClient client) throws Exception {
        connectClient(client, "dummy");
    }

    private void connectClient(MqttClient client, String userName) throws Exception {
        MqttConnectOptions connOpts = new MqttConnectOptions();
        connOpts.setCleanSession(true);
        connOpts.setUserName(userName);
        connOpts.setPassword("12345".toCharArray());
        client.connect(connOpts);
    }
}
```

The above code - is suppose to launch, `serverA`, `serverB` - which are shared subscribers to the same topic. The `testClientPublish` is publishing 100 test messages. serverA and serverB is expected to receive estimated half of these messages each.

Paho's mqtt client library matches topic client side before delivering the messages to the subscribed listeners. **Problem** comes when `MqttTopic.isMatched(topicFilter, topic)` doesn't allow the shared topic subscription to be matched. Luckily, `CommsRouter` has a generic callback for undelivered messages.
Fixed above problem by changing the servers's subscription.

```java
@Test
    public void testServerA() throws Exception {
        MqttClient client = getClient(serverUrl, "s1");
        connectClient(client);

        client.setCallback(new MqttCallback() {
            @Override
            public void connectionLost(Throwable cause) {
                System.out.println("connectionLost");
            }

            @Override
            public void messageArrived(String topic, MqttMessage message) throws Exception {
                System.out.println("message arrived on`` serverA topic: " + topic + " message: " + IOUtils.toString(message.getPayload()));
            }

            @Override
            public void deliveryComplete(IMqttDeliveryToken token) {
                System.out.println("deliveryComplete");
            }
        });

        client.subscribe("$queue//test/+/topic");
        System.out.println("serverA connected");
        Thread.sleep(TimeUnit.HOURS.toMillis(1));
    }

    @Test
    public void testServerB() throws Exception {
        MqttClient client = getClient(serverUrl, "s2");
        connectClient(client);

        client.setCallback(new MqttCallback() {
            @Override
            public void connectionLost(Throwable cause) {
                System.out.println("connectionLost");
            }

            @Override
            public void messageArrived(String topic, MqttMessage message) throws Exception {
                System.out.println("message arrived on`` serverB topic: " + topic + " message: " + IOUtils.toString(message.getPayload()));
            }

            @Override
            public void deliveryComplete(IMqttDeliveryToken token) {
                System.out.println("deliveryComplete");
            }
        });

        client.subscribe("$queue//test/+/topic");
        System.out.println("serverB connected");
        Thread.sleep(TimeUnit.HOURS.toMillis(1));
    }

```

This works now. These messages are received as expected.

## EMQCallback Router

Based on above observation, I added EMQ callback router, to be able to subscribe more than one topic and have its listener support. Usage:
```java
MqttClient client = new MqttClient();
    MqttConnectOptions connOpts = new MqttConnectOptions(); //the connect opt
    client.connect(connOpts);

    String sharedTopic = "$shared/group1/test/topic";
    Map<String, IMqttMessageListener> listeners = new HashMap<>();
    listeners.put(sharedTopic, new IMqttMessageListener() {
        @Override
        public void messageArrived(String topic, MqttMessage message) throws Exception {
          //message received
        }
    });
    //add more topics and listeners in the map.

    client.setCallback(new SharedSubCallbackRouter(listeners));
    client.subscribe(sharedTopic); //Subscribe all via listeners.keySet()
```

More details & source code for `SharedSubCallbackRouter` available on [https://github.com/yogin16/paho-shared-sub-example](https://github.com/yogin16/paho-shared-sub-example)

### References:
1. [http://www.hivemq.com/blog/mqtt-client-load-balancing-with-shared-subscriptions/](http://www.hivemq.com/blog/mqtt-client-load-balancing-with-shared-subscriptions/)
1. [http://emqtt.io/docs/v2/advanced.html](http://emqtt.io/docs/v2/advanced.html)
1. [https://github.com/emqtt/emqttd/issues/639](https://github.com/emqtt/emqttd/issues/639)
1. [https://github.com/codeasone/shared-subscriptions-issue](https://github.com/codeasone/shared-subscriptions-issue)
