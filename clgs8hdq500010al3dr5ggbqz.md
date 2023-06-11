---
title: "Implement WebSocket communication with RabbitMQ"
datePublished: Sat Apr 22 2023 17:07:44 GMT+0000 (Coordinated Universal Time)
cuid: clgs8hdq500010al3dr5ggbqz
slug: implement-websocket-communication-with-rabbitmq
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682196196281/482ff595-a4b3-46f5-86f8-857faca67037.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1682196219694/39f17f8d-81b4-4a46-812f-f1b4eb95e187.png
tags: spring, websockets, java, reactjs, springboot

---

### Introduction

Real-time communication between client and server applications has become an essential feature in modern web applications. WebSocket is a communication protocol that enables real-time, bidirectional communication between a client and a server over a single, long-lived connection. In this article, we'll explore how to implement WebSocket communication between a client app (React as an example) and a Java Spring Boot server application using RabbitMQ as the message broker.

### What are WebSockets?

WebSocket is a protocol that enables full-duplex communication channels over a single TCP connection. Unlike HTTP, which is a request-response-based protocol, WebSockets allow continuous and simultaneous communication between the client and the server. This is particularly useful for applications that require real-time updates, such as chat applications, online gaming, and live data feeds.

### What is RabbitMQ?

RabbitMQ is a robust and scalable open-source message broker that facilitates the exchange of messages between applications through various protocols, including AMQP, STOMP, and MQTT. It supports message routing, message persistence, and various other features that help in building reliable and scalable applications. RabbitMQ acts as a middleman between the client and server applications, ensuring that messages are delivered even if one of the parties is temporarily unavailable.

### Implementation of WebSocket server in Java Spring Boot

To implement WebSocket communication in a Java Spring Boot application, we'll use the Spring Boot Starter WebSocket dependency and the STOMP messaging protocol. Here's a simple example:

Step 1: Add the necessary dependencies to your `pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<dependencies>
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-amqp</artifactId>
   </dependency>
</dependencies>
```

Step 2: Configure RabbitMQ in your [`application.properties`](http://application.properties) or `application.yml`:

```apache
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

Step 3: Create a configuration class to set up the RabbitMQ template and message converter:

```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(jsonMessageConverter());
        return rabbitTemplate;
    }

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

Step 4: Create a service class to send messages to RabbitMQ:

```java

@Service
public class MessageSenderService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendMessage(String exchange, String routingKey, String message) {
        rabbitTemplate.convertAndSend(exchange, routingKey, message);
    }
}
```

Step 5: Create a simple controller to trigger message sending:

```java

@RestController
@RequestMapping("/api")
public class MessageController {

    @Autowired
    private MessageSenderService messageSenderService;

    @PostMapping("/send")
    public ResponseEntity<String> sendMessage(@RequestBody String message) {
        messageSenderService.sendMessage("amq.topic", "messages", message);
        return ResponseEntity.ok("Message sent: " + message);
    }
}
```

With these changes, the Java Spring Boot application is now set up as a server that pushes messages to RabbitMQ. You can send messages to the RabbitMQ exchange by making a POST request to the `/api/send` endpoint with a message payload. The message will be routed to the appropriate queue based on the routing key provided.

You can then configure your WebSocket client (e.g., React application) to subscribe to the RabbitMQ queue and consume messages in real-time.

### Implementation of Client in React app

To implement WebSocket communication in a React application, we'll use the SockJS and STOMP libraries. Here's a simple example:

Step 1: Install the necessary dependencies:

```bash
npm install amqplib
```

Step 2: Create a WebSocket client component:

```javascript

import React, { useEffect, useState } from 'react';
import { connect } from 'amqplib';

const RabbitMQConsumer = () => {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    const connectToRabbitMQ = async () => {
      try {
        const connection = await connect('amqp://guest:guest@localhost:5672');
        const channel = await connection.createChannel();

        const exchange = 'amq.topic';
        const routingKey = 'messages';
        const queueName = 'my-queue';

        await channel.assertQueue(queueName, { durable: false });
        await channel.bindQueue(queueName, exchange, routingKey);

        channel.consume(queueName, (msg) => {
          if (msg !== null) {
            const message = msg.content.toString();
            console.log('Received:', message);
            setMessages((prevMessages) => [...prevMessages, message]);
            channel.ack(msg);
          }
        });
      } catch (error) {
        console.error('Error connecting to RabbitMQ:', error);
      }
    };

    connectToRabbitMQ();

  }, []);

  return (
    <div>
      <h2>Received Messages:</h2>
      <ul>
        {messages.map((message, index) => (
          <li key={index}>{message}</li>
        ))}
      </ul>
    </div>
  );
};

export default RabbitMQConsumer;
```

This example demonstrates the basic setup required to consume messages from a RabbitMQ topic in a React application. The `RabbitMQConsumer` component connects to the RabbitMQ server, creates a channel, and binds a queue to the specified topic with the given routing key. It then consumes messages from the queue and updates the component state with the received messages.

To use this component in your React application, simply import it and include it in your main application component, like this:

```javascript
import React from 'react';
import RabbitMQConsumer from './RabbitMQConsumer';

function App() {
  return (
    <div className="App">
      <RabbitMQConsumer />
    </div>
  );
}

export default App;
```

Now your React application is set up to consume messages from the RabbitMQ topic and display them in real-time.

### Conclusion

In this guide, we demonstrated how to implement real-time communication between a React application and a Java Spring Boot application using RabbitMQ as a message broker. By following the examples provided, you can implement real-time communication between a React application and a Java Spring Boot application using RabbitMQ. This setup allows you to build scalable, reliable, and real-time applications for a variety of use cases, such as chat applications, live data feeds, and online gaming.