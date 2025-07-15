# Load Balancing

## 📋 Overview

Distribute incoming network traffic across multiple backend servers to ensure high availability, fault tolerance, and optimal resource utilization while preventing any single server from becoming overwhelmed.

## 🎯 Problem Statement

As application traffic grows, a single server becomes a bottleneck and single point of failure. How do you distribute requests across multiple servers while maintaining session consistency, handling server failures gracefully, and optimizing response times?

## 💡 Solution

Deploy load balancers to intelligently distribute traffic using various algorithms (round-robin, least connections, weighted) while monitoring server health and automatically routing around failures.

## 🏗️ Architecture

### Components

- **Load Balancer**: Entry point that distributes incoming requests
- **Backend Server Pool**: Multiple application servers handling requests
- **Health Check System**: Monitors server availability and performance
- **Session Store**: External session management for sticky sessions
- **Monitoring Dashboard**: Real-time metrics and alerting

### Flow

1. **Request Reception**: Client request hits the load balancer
2. **Algorithm Selection**: Load balancer chooses backend server using configured algorithm
3. **Health Verification**: Ensures selected server is healthy and responsive
4. **Request Forwarding**: Routes request to chosen backend server
5. **Response Handling**: Returns backend response to client

## 📝 Implementation

### NGINX Load Balancer Configuration

nginx

```nginx
# Basic load balancing configuration
upstream backend_servers {
    # Load balancing method
    least_conn;
    
    # Backend servers with weights
    server 10.0.1.10:8080 weight=3;
    server 10.0.1.11:8080 weight=2;
    server 10.0.1.12:8080 weight=1;
    server 10.0.1.13:8080 backup;
    
    # Health check settings
    keepalive 32;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

### HAProxy Configuration

yaml

```yaml
# HAProxy load balancer setup
global:
  daemon
  maxconn 4096

defaults:
  mode http
  timeout connect 5s
  timeout client 50s
  timeout server 50s
  option httpchk GET /health

frontend web_frontend
  bind *:80
  default_backend web_servers

backend web_servers
  balance roundrobin
  option httpchk
  server web1 10.0.1.10:8080 check weight 100
  server web2 10.0.1.11:8080 check weight 100
  server web3 10.0.1.12:8080 check weight 50
```

### Application-Level Load Balancing

javascript

```javascript
// Client-side load balancing with retry logic
class LoadBalancer {
    constructor(servers) {
        this.servers = servers;
        this.currentIndex = 0;
        this.failedServers = new Set();
    }
    
    async request(path, options = {}) {
        const maxRetries = this.servers.length;
        
        for (let retry = 0; retry < maxRetries; retry++) {
            const server = this.getNextServer();
            
            try {
                const response = await fetch(`${server}${path}`, {
                    ...options,
                    timeout: 5000
                });
                
                // Remove from failed list on success
                this.failedServers.delete(server);
                return response;
                
            } catch (error) {
                console.warn(`Server ${server} failed:`, error.message);
                this.failedServers.add(server);
            }
        }
        
        throw new Error('All servers unavailable');
    }
    
    getNextServer() {
        // Round-robin with health checking
        const availableServers = this.servers.filter(
            server => !this.failedServers.has(server)
        );
        
        if (availableServers.length === 0) {
            // Reset failed servers if all are down
            this.failedServers.clear();
            return this.servers[0];
        }
        
        const server = availableServers[this.currentIndex % availableServers.length];
        this.currentIndex++;
        return server;
    }
}
```

## ⚖️ Trade-offs

### Pros

- ✅ **High availability** through redundancy and failover
- ✅ **Scalability** by adding more backend servers
- ✅ **Performance optimization** via intelligent request distribution
- ✅ **Fault tolerance** automatic routing around failed servers
- ✅ **Flexibility** supports multiple load balancing algorithms
- ✅ **SSL termination** centralized certificate management

### Cons

- ❌ **Single point of failure** if load balancer isn't redundant
- ❌ **Added latency** from additional network hop
- ❌ **Session complexity** managing state across multiple servers
- ❌ **Configuration overhead** tuning algorithms and health checks
- ❌ **Cost increase** additional infrastructure and licensing

## 🔧 When to Use

- **High traffic applications** requiring horizontal scaling
- **Mission-critical systems** needing fault tolerance
- **Microservices architectures** with multiple service instances
- **Global applications** requiring geographic load distribution
- **Auto-scaling environments** with dynamic server pools
- **SSL termination** centralized certificate management needs

## 🚫 When to Avoid

- **Single server applications** with low traffic volumes
- **Stateful applications** heavily dependent on server-side sessions
- **Simple prototypes** where complexity outweighs benefits
- **Cost-sensitive projects** where additional infrastructure isn't justified
- **Applications with strict latency requirements** where additional hops matter

## 🌍 Real-world Examples

### Netflix

Uses Eureka for service discovery and Ribbon for client-side load balancing across thousands of microservice instances globally.

### Amazon ELB

Elastic Load Balancing automatically distributes traffic across multiple EC2 instances with health checks and auto-scaling integration.

### Cloudflare

Global load balancing with geographic routing, DDoS protection, and intelligent failover across multiple data centers.

## 📚 Related Patterns

- [API Gateways](https://claude.ai/web-patterns/api-gateways/)
- [Circuit Breakers](https://claude.ai/microservices/circuit-breakers/)
- [Service Discovery](https://claude.ai/microservices/service-discovery/)
- [Caching Strategies](https://claude.ai/data-patterns/caching-strategies/)

## 🔗 References

- [NGINX Load Balancing Guide](https://nginx.org/en/docs/http/load_balancing.html)
- [HAProxy Documentation](https://www.haproxy.org/download/1.8/doc/configuration.txt)
- [AWS ELB Best Practices](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/best-practices.html)
- [Load Balancing Algorithms Explained](https://kemptechnologies.com/load-balancer/load-balancing-algorithms-techniques/)

---

_Pattern documented using [pattern template](https://claude.ai/templates/pattern-template.md)_