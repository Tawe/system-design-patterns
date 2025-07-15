# Real-Time Chat System

## 📋 Overview

A scalable real-time messaging system that enables users across multiple websites/applications to communicate instantly with built-in moderation, high availability, and sub-100ms latency.

## 🎯 Problem Statement

Design a messaging system that allows users across multiple websites/applications to communicate in real-time with minimal latency, while ensuring message moderation and system reliability. The system must handle millions of concurrent users, maintain message ordering, and provide cross-platform compatibility.

## 💡 Solution

A distributed architecture using WebSocket gateways, message queues, and asynchronous moderation to achieve real-time communication without sacrificing reliability or content safety.

## 🏗️ Architecture

```
[Website A] ←→ [Load Balancer] ←→ [WebSocket Gateway Cluster] ←→ [Message Queue]
[Website B] ←→                                                      ↓
                                                            [Message Service]
                                                                    ↓
                                                            [Moderation Service]
                                                                    ↓
                                                            [Broadcast Service]
                                                                    ↓
                                                            [Cache & Database]
```

## Components

### 1. Client Layer

- **Technology**: React/Vue.js with WebSocket client
- **Responsibility**: UI, connection management, message display
- **Key Features**:
    - Auto-reconnection on connection loss
    - Message queuing during offline periods
    - Typing indicators and read receipts

### 2. Load Balancer

- **Technology**: NGINX/HAProxy with WebSocket proxy support
- **Responsibility**: Distribute connections across gateway instances
- **Configuration**:
    - Sticky sessions for WebSocket connections
    - Health checks for gateway instances
    - SSL termination

### 3. WebSocket Gateway Cluster

- **Technology**: Node.js with Socket.io or Go with Gorilla WebSocket
- **Responsibility**: Manage real-time connections
- **Key Features**:
    - Connection state management
    - Message routing
    - Rate limiting per connection
    - Authentication validation

### 4. Connection Manager

- **Technology**: Redis Cluster
- **Responsibility**: Track active connections across gateway instances
- **Data Structure**:

json

```json
{
  "user:123": {
    "gateway": "gateway-2",
    "socketId": "abc123",
    "lastSeen": "2025-07-14T10:30:00Z"
  }
}
```

### 5. Message Queue

- **Technology**: Apache Kafka or RabbitMQ
- **Responsibility**: Reliable message delivery and ordering
- **Benefits**:
    - Durability during service outages
    - Message replay capability
    - Horizontal scaling

### 6. Message Service

- **Technology**: Microservice (Go/Node.js/Java)
- **Responsibility**: Message processing and persistence
- **Features**:
    - Message validation
    - Duplicate detection
    - Metadata enrichment (timestamps, user info)

### 7. Moderation Service

- **Technology**: AI/ML service with rule engine
- **Responsibility**: Content filtering without blocking message flow
- **Approach**:
    - **Async moderation** - messages deliver immediately
    - Post-delivery content analysis
    - Retroactive message removal/flagging
- **Features**:
    - Profanity filtering
    - Spam detection
    - Image/video content analysis

### 8. Broadcast Service

- **Technology**: Pub/Sub system
- **Responsibility**: Fan-out messages to all connected clients
- **Optimization**:
    - Channel-based routing (rooms/topics)
    - Message batching for efficiency
    - Rate limiting to prevent spam

### 9. Data Layer

- **Message Database**: PostgreSQL (sharded by chat room)
- **Cache**: Redis for recent messages and user sessions
- **User Database**: MongoDB for user profiles and preferences
- **Analytics**: ClickHouse for message metrics and insights

## Performance Characteristics

|Metric|Target|Implementation|
|---|---|---|
|Message Latency|< 100ms|WebSocket + Redis|
|Concurrent Users|100K+ per gateway|Horizontal scaling|
|Message Throughput|10K msgs/sec|Kafka partitioning|
|Availability|99.9%|Multi-region deployment|
|Moderation Speed|< 500ms|Async processing|

## Scaling Strategies

### Horizontal Scaling

- **Gateway Instances**: Auto-scale based on connection count
- **Message Services**: Scale based on queue depth
- **Database**: Shard by chat room or user ID

### Vertical Optimizations

- Connection pooling in gateways
- Message batching in queues
- Read replicas for message history

### Global Distribution

- Regional gateway clusters
- Message replication across regions
- CDN for static assets and file sharing

## Trade-offs

### Pros

✅ **Real-time performance** - Sub-100ms message delivery  
✅ **High availability** - No single points of failure  
✅ **Scalable** - Handles millions of concurrent users  
✅ **Reliable** - Message queue ensures delivery  
✅ **Cross-platform** - Works across multiple websites

### Cons

❌ **Complexity** - Multiple services to manage  
❌ **Cost** - Requires significant infrastructure  
❌ **Consistency** - Eventually consistent across regions  
❌ **Debugging** - Distributed system challenges

## Implementation Considerations

### Message Ordering

javascript

```javascript
// Message structure with ordering
{
  "id": "uuid",
  "chatRoom": "room-123",
  "userId": "user-456", 
  "content": "Hello world",
  "timestamp": "2025-07-14T10:30:00Z",
  "sequence": 12345,
  "type": "text"
}
```

### Connection Resilience

javascript

```javascript
// Client-side reconnection logic
class ChatClient {
  connect() {
    this.socket = new WebSocket(this.endpoint);
    this.socket.onclose = () => {
      setTimeout(() => this.connect(), this.backoffMs);
      this.backoffMs = Math.min(this.backoffMs * 2, 30000);
    };
  }
}
```

### Rate Limiting

javascript

```javascript
// Redis-based rate limiting
const rateLimit = async (userId, limit = 10, window = 60) => {
  const key = `rate_limit:${userId}:${Math.floor(Date.now() / (window * 1000))}`;
  const current = await redis.incr(key);
  if (current === 1) await redis.expire(key, window);
  return current <= limit;
};
```

## Alternative Approaches

### 1. Simpler WebSocket-Only

- **Use Case**: Small to medium applications
- **Trade-off**: Less reliability, harder to scale
- **When to Use**: MVP or single-region deployment

### 2. Firebase/Pusher SaaS

- **Use Case**: Rapid prototyping
- **Trade-off**: Vendor lock-in, less control
- **When to Use**: Early stage or non-critical applications

### 3. Server-Sent Events (SSE)

- **Use Case**: One-way communication needs
- **Trade-off**: No bidirectional communication
- **When to Use**: Notifications or live updates only

## Monitoring & Observability

### Key Metrics

- Connection count per gateway
- Message latency (end-to-end)
- Queue depth and processing rate
- Moderation accuracy and speed
- Error rates by service

### Alerts

- High message latency (> 500ms)
- Gateway connection failures
- Queue backlog growth
- Moderation service downtime

## Example Use Cases

- **Multi-tenant chat platforms** (Slack, Discord)
- **Customer support systems** (Intercom, Zendesk)
- **Gaming communication** (in-game chat)
- **Live streaming chat** (Twitch, YouTube)
- **Collaborative tools** (Figma comments, Google Docs)

---

_This pattern is optimized for high-scale, real-time communication with built-in moderation and cross-platform support._