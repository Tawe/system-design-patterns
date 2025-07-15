# Message Queues

## 📋 Overview

Enable asynchronous communication between distributed system components using message queues to decouple producers and consumers, ensuring reliable message delivery and system resilience.

## 🎯 Problem Statement

How do you handle communication between services when they operate at different speeds, may be temporarily unavailable, or need to process large volumes of data without blocking each other? Direct service-to-service calls create tight coupling and can lead to cascading failures.

## 💡 Solution

Implement message queues as intermediaries that store and forward messages between producers and consumers, enabling asynchronous processing, load leveling, and fault tolerance.

## 🏗️ Architecture

### Components

- **Message Producer**: Services that send messages to queues
- **Message Queue/Broker**: Stores and manages message delivery (Kafka, RabbitMQ, SQS)
- **Message Consumer**: Services that process messages from queues
- **Dead Letter Queue**: Handles failed message processing
- **Monitoring System**: Tracks queue metrics and consumer health

### Flow

1. **Message Production**: Producer sends message to queue with routing information
2. **Queue Storage**: Message broker stores message with metadata and ordering
3. **Consumer Discovery**: Available consumers poll or receive pushed messages
4. **Message Processing**: Consumer processes message and acknowledges completion
5. **Cleanup/Retry**: Successfully processed messages removed, failed messages retried or sent to DLQ

## 📝 Implementation

### Apache Kafka Implementation

javascript

```javascript
// Kafka Producer
const kafka = require('kafkajs');

class OrderEventProducer {
    constructor() {
        this.kafka = kafka({
            clientId: 'order-service',
            brokers: ['kafka1:9092', 'kafka2:9092']
        });
        this.producer = this.kafka.producer();
    }
    
    async publishOrderCreated(order) {
        try {
            await this.producer.send({
                topic: 'order-events',
                partition: order.customerId % 3, // Partition by customer
                messages: [{
                    key: order.id,
                    value: JSON.stringify({
                        eventType: 'ORDER_CREATED',
                        orderId: order.id,
                        customerId: order.customerId,
                        timestamp: new Date().toISOString(),
                        data: order
                    })
                }]
            });
        } catch (error) {
            console.error('Failed to publish order event:', error);
            throw error;
        }
    }
}

// Kafka Consumer
class OrderEventConsumer {
    constructor() {
        this.kafka = kafka({
            clientId: 'notification-service',
            brokers: ['kafka1:9092', 'kafka2:9092']
        });
        this.consumer = this.kafka.consumer({ 
            groupId: 'notification-group' 
        });
    }
    
    async start() {
        await this.consumer.subscribe({ topic: 'order-events' });
        
        await this.consumer.run({
            eachMessage: async ({ topic, partition, message }) => {
                try {
                    const event = JSON.parse(message.value.toString());
                    await this.processOrderEvent(event);
                } catch (error) {
                    console.error('Failed to process message:', error);
                    // Send to dead letter queue
                    await this.sendToDeadLetterQueue(message);
                }
            }
        });
    }
    
    async processOrderEvent(event) {
        if (event.eventType === 'ORDER_CREATED') {
            await this.sendOrderConfirmationEmail(event.data);
        }
    }
}
```

### RabbitMQ Configuration

yaml

```yaml
# RabbitMQ setup with exchanges and queues
rabbitmq:
  exchanges:
    - name: order_exchange
      type: topic
      durable: true
    - name: notification_exchange  
      type: direct
      durable: true
      
  queues:
    - name: order_processing_queue
      durable: true
      arguments:
        x-message-ttl: 300000  # 5 minutes
        x-dead-letter-exchange: order_dlx
        
    - name: email_notification_queue
      durable: true
      auto_delete: false
      
    - name: order_dlq
      durable: true
      
  bindings:
    - exchange: order_exchange
      queue: order_processing_queue
      routing_key: "order.created"
      
    - exchange: notification_exchange
      queue: email_notification_queue
      routing_key: "email"
```

### AWS SQS with Retry Logic

javascript

```javascript
// SQS Producer with error handling
const AWS = require('aws-sdk');
const sqs = new AWS.SQS({ region: 'us-east-1' });

class SQSMessageProducer {
    constructor(queueUrl) {
        this.queueUrl = queueUrl;
    }
    
    async sendMessage(messageBody, delaySeconds = 0) {
        const params = {
            QueueUrl: this.queueUrl,
            MessageBody: JSON.stringify(messageBody),
            DelaySeconds: delaySeconds,
            MessageAttributes: {
                'RetryCount': {
                    DataType: 'Number',
                    StringValue: '0'
                },
                'OriginalTimestamp': {
                    DataType: 'String', 
                    StringValue: new Date().toISOString()
                }
            }
        };
        
        try {
            const result = await sqs.sendMessage(params).promise();
            return result.MessageId;
        } catch (error) {
            console.error('SQS send failed:', error);
            throw error;
        }
    }
}

// SQS Consumer with exponential backoff
class SQSMessageConsumer {
    constructor(queueUrl, deadLetterQueueUrl) {
        this.queueUrl = queueUrl;
        this.deadLetterQueueUrl = deadLetterQueueUrl;
        this.maxRetries = 3;
    }
    
    async pollMessages() {
        const params = {
            QueueUrl: this.queueUrl,
            MaxNumberOfMessages: 10,
            WaitTimeSeconds: 20, // Long polling
            VisibilityTimeout: 30
        };
        
        try {
            const data = await sqs.receiveMessage(params).promise();
            
            if (data.Messages) {
                await Promise.all(
                    data.Messages.map(message => this.processMessage(message))
                );
            }
        } catch (error) {
            console.error('Polling failed:', error);
        }
        
        // Continue polling
        setTimeout(() => this.pollMessages(), 1000);
    }
    
    async processMessage(message) {
        try {
            const body = JSON.parse(message.Body);
            const retryCount = parseInt(
                message.MessageAttributes?.RetryCount?.StringValue || '0'
            );
            
            // Process business logic
            await this.handleBusinessLogic(body);
            
            // Delete message on success
            await this.deleteMessage(message.ReceiptHandle);
            
        } catch (error) {
            await this.handleProcessingError(message, error);
        }
    }
    
    async handleProcessingError(message, error) {
        const retryCount = parseInt(
            message.MessageAttributes?.RetryCount?.StringValue || '0'
        );
        
        if (retryCount < this.maxRetries) {
            // Retry with exponential backoff
            const delaySeconds = Math.pow(2, retryCount) * 5;
            await this.requeueWithDelay(message, delaySeconds, retryCount + 1);
        } else {
            // Send to dead letter queue
            await this.sendToDeadLetterQueue(message, error);
        }
        
        // Always delete from original queue
        await this.deleteMessage(message.ReceiptHandle);
    }
}
```

## ⚖️ Trade-offs

### Pros

- ✅ **Loose coupling** between services through async communication
- ✅ **Fault tolerance** messages persist during consumer downtime
- ✅ **Load leveling** smooths out traffic spikes and processing bursts
- ✅ **Scalability** consumers can scale independently of producers
- ✅ **Reliability** guaranteed delivery with acknowledgments and retries
- ✅ **Flexibility** multiple consumers can process same message stream

### Cons

- ❌ **Complexity** additional infrastructure to manage and monitor
- ❌ **Eventual consistency** introduces delays in data propagation
- ❌ **Message ordering** can be challenging across multiple consumers
- ❌ **Debugging difficulty** tracing messages across async boundaries
- ❌ **Duplicate processing** consumers must handle at-least-once delivery
- ❌ **Storage overhead** queue persistence and backup requirements

## 🔧 When to Use

- **Microservices architectures** requiring loose coupling
- **High-volume data processing** with variable processing speeds
- **Event-driven systems** with publish-subscribe patterns
- **Background job processing** (image processing, email sending)
- **System integration** between different applications or vendors
- **Load balancing** across multiple worker instances

## 🚫 When to Avoid

- **Real-time synchronous** operations requiring immediate responses
- **Simple applications** where direct calls are sufficient
- **Strong consistency requirements** where eventual consistency isn't acceptable
- **Low-latency systems** where message overhead is prohibitive
- **Small-scale systems** where queue overhead exceeds benefits

## 🌍 Real-world Examples

### Netflix

Uses Apache Kafka for real-time data streaming, event sourcing, and microservice communication across their global streaming platform.

### Uber

Implements message queues for ride matching, payment processing, and real-time location tracking with millions of messages per second.

### Airbnb

Uses RabbitMQ and Amazon SQS for booking workflows, payment processing, and notification systems with retry mechanisms.

## 📚 Related Patterns

- [Event Sourcing](https://claude.ai/data-patterns/event-sourcing/)
- [Circuit Breakers](https://claude.ai/microservices/circuit-breakers/)
- [Real-Time Chat](https://claude.ai/messaging-systems/real-time-chat/)
- [Microservices Communication](https://claude.ai/microservices/service-discovery/)

## 🔗 References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
- [AWS SQS Best Practices](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-best-practices.html)
- [Message Queue Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/)

---

_Pattern documented using [pattern template](https://claude.ai/templates/pattern-template.md)_