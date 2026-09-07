# Production-Style Kubernetes Ingress Architecture
## Exposing Multiple Microservices Through AWS Application Load Balancer

![Kubernetes Ingress with AWS ALB Architecture](./k8s_ingress_alb_hero.jpg)

**DevOps Engineering Presentation & Architecture Deep Dive**  
**Author / Presenter:** Senior DevOps Engineer  
**Scope:** Kubernetes Ingress, AWS ALB Controller, ClusterIP Services, Workload Pods, and Asynchronous Microservice Pipelines  
**Source of Truth:** Repository Kubernetes YAML Manifests (`1-voting-app.yaml` through `miniproject-ingress.yaml`)  
**Target Domains:** `vote.somesh.xyz` & `result.somesh.xyz`

---

## 📑 Repository Manifests & File Index

| File | Type | Description |
| :--- | :--- | :--- |
| **[`miniproject-ingress.yaml`](./miniproject-ingress.yaml)** | `Ingress` | Layer-7 Ingress definition with AWS ALB annotations (`internet-facing`, `target-type: ip`) |
| **[`1-voting-app.yaml`](./1-voting-app.yaml)** | `Deployment` | Voting App frontend (Python/Flask, 2 replicas, port 80) |
| **[`2-volting-app-service.yaml`](./2-volting-app-service.yaml)** | `Service` | ClusterIP Service for Voting App (Port 80) |
| **[`3-redics-app.yaml`](./3-redics-app.yaml)** | `Deployment` | In-Memory Redis queue (1 replica, port 6379) |
| **[`4-redics-service.yaml`](./4-redics-service.yaml)** | `Service` | ClusterIP Service for Redis (`redis:6379`) |
| **[`5-worker-deploy.yaml`](./5-worker-deploy.yaml)** | `Deployment` | Background queue consumer (.NET Core Daemon) |
| **[`6-postgres-deploy.yaml`](./6-postgres-deploy.yaml)** | `Deployment` | PostgreSQL database (Port 5432) |
| **[`7-postgres-service.yaml`](./7-postgres-service.yaml)** | `Service` | ClusterIP Service for PostgreSQL (`db:5432`) |
| **[`8-result-app-deployment.yaml`](./8-result-app-deployment.yaml)** | `Deployment` | Result App dashboard (Node.js/Angular, port 80) |
| **[`9-result-service.yaml`](./9-result-service.yaml)** | `Service` | ClusterIP Service for Result App (Port 80) |
| **[`presentation.html`](./presentation.html)** | `Slide Deck` | Interactive dark-mode presentation with speaker notes drawer (`open presentation.html`) |
| **[`KUBERNETES_INGRESS_PRESENTATION.md`](./KUBERNETES_INGRESS_PRESENTATION.md)** | `Document` | Standalone presentation document copy |

---

## 🏛️ Architecture Blueprint

```mermaid
flowchart TD
    subgraph PublicInternet[" Public Internet / Clients "]
        User1[" Voter Client\n(Browser / Mobile) "]
        User2[" Analytics Client\n(Browser) "]
    end

    subgraph AWSCloud[" AWS Cloud Infrastructure "]
        Route53[" Route 53 DNS\n(vote.somesh.xyz & result.somesh.xyz) "]
        
        subgraph AWSALB[" AWS Application Load Balancer (ALB) "]
            ALBListener[" HTTP Listener: Port 80 "]
            ALBRule1[" Host Header Rule:\nvote.somesh.xyz "]
            ALBRule2[" Host Header Rule:\nresult.somesh.xyz "]
            TG1[" Target Group 1\n(Pod IP Target Mode) "]
            TG2[" Target Group 2\n(Pod IP Target Mode) "]
        end
    end

    subgraph K8sCluster[" AWS EKS / Kubernetes Cluster "]
        subgraph IngressCtrl[" Kubernetes Ingress Controller Layer "]
            ALBController[" AWS Load Balancer Controller\n(Watches Ingress Resources) "]
            K8sIngress[" Ingress Resource:\nvoting-ingress "]
        end

        subgraph SvcLayer[" ClusterIP Service Layer (Internal Discovery) "]
            VoteSvc[" Service: voting-service\nClusterIP: 10.100.x.x:80 "]
            ResultSvc[" Service: result-service\nClusterIP: 10.100.x.y:80 "]
            RedisSvc[" Service: redis\nClusterIP: 10.100.x.z:6379 "]
            DBSvc[" Service: db\nClusterIP: 10.100.x.w:5432 "]
        end

        subgraph PodLayer[" Workload Pods Layer (VPC CNI Pod IPs) "]
            VotePod1[" Pod: voting-app-pod-1\nIP: 10.0.1.15:80 "]
            VotePod2[" Pod: voting-app-pod-2\nIP: 10.0.2.42:80 "]
            ResultPod[" Pod: result-app-pod-1\nIP: 10.0.2.88:80 "]
            WorkerPod[" Pod: worker-app-pod\n(Background Processor) "]
            RedisPod[" Pod: redis-pod\n(In-Memory Queue) "]
            PostgresPod[" Pod: postgres-pod\n(Relational Storage) "]
        end
    end

    %% External Traffic Connections
    User1 -->|"1. HTTP GET/POST vote.somesh.xyz"| Route53
    User2 -->|"1. HTTP GET result.somesh.xyz"| Route53
    Route53 -->|"CNAME / Alias"| AWSALB
    AWSALB --> ALBListener
    ALBListener --> ALBRule1
    ALBListener --> ALBRule2
    ALBRule1 --> TG1
    ALBRule2 --> TG2

    %% Ingress Controller reconciliation
    K8sIngress -.->|"Reconciled by"| ALBController
    ALBController -.->|"Configures Listeners & TGs"| AWSALB

    %% Direct Pod Routing via target-type: ip
    TG1 ===>|"Direct Route (Bypasses NodePort/kube-proxy)"| VotePod1
    TG1 ===>|"Direct Route"| VotePod2
    TG2 ===>|"Direct Route"| ResultPod

    %% Internal Data Pipeline
    VotePod1 -->|"Pushes Vote (TCP 6379)"| RedisSvc
    VotePod2 -->|"Pushes Vote (TCP 6379)"| RedisSvc
    RedisSvc --> RedisPod
    WorkerPod -->|"Pulls raw vote via BLPOP"| RedisSvc
    WorkerPod -->|"Inserts / Updates Vote (TCP 5432)"| DBSvc
    DBSvc --> PostgresPod
    ResultPod -->|"Queries Aggregated Votes (TCP 5432)"| DBSvc
```

---

## 🖥️ Interactive Presentation
You can run the bundled interactive HTML slide deck locally in your browser:
```bash
open presentation.html
```
* **Navigate:** Press Left (`←`) and Right (`→`) arrow keys.
* **Speaker Notes:** Press `S` to toggle presenter notes.
* **Interview Q&A:** Click the **📋 20 Interview Q&As** button in the header.

---

# 14-Slide Technical Presentation Deck

---

## Slide 1 — Title
### Kubernetes Ingress Architecture: Exposing Multiple Microservices Through AWS Application Load Balancer
**DevOps Project Presentation** | **Kubernetes** | **AWS EKS** | **Ingress** | **Microservices**

#### Technical Highlights
* **Centralized Edge Ingestion:** Unifying external traffic for distributed microservices through a single managed AWS Application Load Balancer (ALB).
* **Host-Based Layer 7 Switching:** Dynamic routing for `vote.somesh.xyz` and `result.somesh.xyz` at the cloud boundary.
* **Direct Pod IP Ingress:** Native AWS VPC CNI routing via `target-type: ip`, bypassing NodePort and kube-proxy DNAT latency.
* **Zero-Trust Internal Network:** Application workloads, cache, and state stores isolated entirely on private Kubernetes `ClusterIP` networks.

> 🎙️ **Speaker Notes:**  
> *"Good morning, everyone. Today I'm presenting a production-style Kubernetes Ingress architecture based on a real-world microservice deployment on AWS. When designing containerized systems, one of the most critical platform decisions is how external user traffic enters the cluster safely and reaches target pods. In this project, I engineered a centralized ingress pattern using the AWS Load Balancer Controller and Kubernetes Ingress, exposing multiple microservices—our voting frontend and real-time results portal—through a single managed Application Load Balancer using host-based routing. Over the next 15 minutes, I'll walk you through the end-to-end traffic flow, why this architecture avoids anti-patterns like multiple public load balancers, the AWS-specific annotations configured in the manifests, and how requests journey from DNS down to pod network namespaces and state stores."*

---

## Slide 2 — Project Overview
### Example Voting Application — Multi-Tier Microservice Topology

```mermaid
flowchart LR
    subgraph FrontendTier[" Frontend Tier (Stateless Web) "]
        VoteApp[" Voting App (Python/Flask)\n2 Replicas | Port 80 "]
        ResultApp[" Result App (Node.js/Angular)\n1 Replica | Port 80 "]
    end

    subgraph AsyncQueue[" In-Memory Queue Tier "]
        Redis[" Redis Queue\n1 Replica | Port 6379 "]
    end

    subgraph WorkerTier[" Asynchronous Processing Tier "]
        Worker[" Worker (.NET Core)\n1 Replica | Background Daemon "]
    end

    subgraph PersistenceTier[" Database Tier "]
        Postgres[" PostgreSQL 9.4\n1 Replica | Port 5432 "]
    end

    VoteApp -->|"1. Push Vote JSON"| Redis
    Redis -->|"2. Pop Queued Event"| Worker
    Worker -->|"3. Write / Aggregate Record"| Postgres
    Postgres -->|"4. Poll / Stream Tally"| ResultApp
```

#### Component Breakdown & Functional Roles
1. **Voting App (`dockersamples/examplevotingapp_vote`):** High-throughput user-facing web frontend (Python/Flask). Accepts user votes (`Cats` vs `Dogs`) and immediately enqueues them onto Redis to guarantee sub-millisecond response times. Runs **2 replicas** for high availability.
2. **Redis (`redis:latest`):** In-memory message buffer buffering write surges, decoupling frontend ingestion from backend database writes.
3. **Worker (`dockersamples/examplevotingapp_worker`):** Background daemon written in .NET Core. Continuously monitors the Redis queue, deselects votes, and writes them to PostgreSQL.
4. **PostgreSQL (`postgres:9.4`):** Relational persistence store housing the `votes` schema and running totals.
5. **Result App (`dockersamples/examplevotingapp_result`):** Node.js & Socket.io dashboard querying PostgreSQL to render real-time poll results to viewers.

> 🎙️ **Speaker Notes:**  
> *"To understand our networking requirements, let's look at the application topology. This is a classic distributed microservice architecture featuring an asynchronous processing pipeline. Notice the separation of concerns: our Python voting frontend does not touch the database directly; it offloads votes to Redis in memory. A background .NET worker consumes events from Redis and writes them to PostgreSQL. Finally, the Node.js result app queries Postgres to display real-time analytics. From an infrastructure perspective, notice that out of five components, only TWO require public ingress: the Voting App and the Result App. The Worker, Redis, and PostgreSQL must remain strictly private. This is where Kubernetes Ingress shines."*

---

## Slide 3 — Why Do We Need Ingress?
### The Problem: Exposing Multiple Applications in Kubernetes

```mermaid
flowchart TD
    subgraph AntiPattern[" Anti-Pattern: Individual LoadBalancer Services "]
        direction TB
        ClientA[" Client A "] -->|"TCP"| LB1[" AWS NLB/ALB #1\n($18-25/mo + LCU) "]
        ClientB[" Client B "] -->|"TCP"| LB2[" AWS NLB/ALB #2\n($18-25/mo + LCU) "]
        LB1 --> NodePort1[" NodePort 31080 "] --> Pods1[" Voting Pods "]
        LB2 --> NodePort2[" NodePort 32080 "] --> Pods2[" Result Pods "]
        style AntiPattern fill:#ffebee,stroke:#c62828,stroke-width:2px
    end

    subgraph IngressPattern[" DevOps Best Practice: Unified Kubernetes Ingress "]
        direction TB
        ClientAll[" All Clients\n(*.somesh.xyz) "] -->|"Single HTTP/S Port 80/443"| SingleALB[" Single AWS ALB\n(Unified Cloud Perimeter) "]
        SingleALB --> IngressCtrl2[" Kubernetes Ingress (L7 Switching) "]
        IngressCtrl2 -->|"vote.somesh.xyz"| SvcVote[" voting-service "] --> PodsA[" Voting Pods "]
        IngressCtrl2 -->|"result.somesh.xyz"| SvcRes[" result-service "] --> PodsB[" Result Pods "]
        style IngressPattern fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    end
```

#### Comparison Matrix: Service Type LoadBalancer vs. Kubernetes Ingress

| Architecture Dimension | Multiple `LoadBalancer` Services | Kubernetes Ingress + AWS ALB |
| :--- | :--- | :--- |
| **AWS Resource Footprint** | Provisions a dedicated AWS ELB/NLB per Service | **Single AWS ALB** serves entire cluster ingress |
| **Cloud Cost Model** | Base hourly cost multiplied by $N$ services + data fees | Consolidated base cost; pay only for combined LCUs |
| **TLS / SSL Management** | Certificates attached across multiple load balancers | Centralized ACM wildcard certificate binding on one listener |
| **DNS Architecture** | Disjointed CNAME records to distinct AWS DNS names | Clean subdomain mapping (`*.somesh.xyz`) to a single ALB |
| **Routing Intelligence** | Layer 4 (IP/Port) only; no HTTP header visibility | **Layer 7 aware:** Host headers, URI paths, HTTP methods |
| **Attack Surface** | Multiple public IPs/endpoints exposed to the internet | Single, hardened security perimeter / WAF inspection point |

> 🎙️ **Speaker Notes:**  
> *"When junior engineers deploy multiple web services on Kubernetes, their immediate instinct is often to change every Service type to LoadBalancer. From a DevOps and cloud economics perspective, that is an anti-pattern. If you have 10 microservices, creating 10 LoadBalancer services provisions 10 distinct AWS ELBs or NLBs. That means 10 times the base AWS hourly cost, fragmented DNS records, multiple public endpoints to secure, and zero Layer-7 routing intelligence. Instead, our architecture uses Kubernetes Ingress. A single AWS Application Load Balancer acts as our cluster front door. We route traffic based on HTTP Host headers—sending vote.somesh.xyz to our voting pods and result.somesh.xyz to our result pods—while maintaining internal services as private ClusterIP resources."*

---

## Slide 4 — What Is Kubernetes Ingress?
### Decoupling Routing Policy from Ingress Execution

```mermaid
flowchart TD
    subgraph ControlPlane[" 1. Kubernetes Control Plane (API & Desired State) "]
        IngressYAML[" Ingress Manifest\n(miniproject-ingress.yaml)\nkind: Ingress\nrules: [vote, result] "]
        KubeAPI[" kube-apiserver "]
        IngressYAML -->|"kubectl apply"| KubeAPI
    end

    subgraph DataPlaneReconciliation[" 2. Controller & Reconciliation Engine "]
        AWSController[" AWS Load Balancer Controller Pod\n(Runs inside kube-system / aws-load-balancer-controller) "]
        KubeAPI -.->|"Watch Event (Ingress Created/Modified)"| AWSController
    end

    subgraph CloudInfra[" 3. AWS Managed Infrastructure (Data Plane) "]
        AWSSDK[" AWS API (ELBv2 SDK) "]
        ALBResource[" AWS Application Load Balancer\n- Listener 80\n- Host Rules: vote.* / result.*\n- Target Groups: Pod IPs "]
        AWSController -->|"Describe / Create / Modify"| AWSSDK
        AWSSDK -->|"Configures"| ALBResource
    end
```

#### The Four Core Primitives
1. **The Ingress Resource:** Declarative Kubernetes API object (`networking.k8s.io/v1`) defining routing rules (hostnames, paths, and target backend Services). *It does not route packets on its own!*
2. **The Ingress Controller:** Active controller daemon (**AWS Load Balancer Controller**) watching the Kubernetes API and translating rules into cloud load-balancing infrastructure (AWS ALB).
3. **The Kubernetes Service:** Persistent abstraction providing a stable virtual IP and CoreDNS name for ephemeral pods via label selectors.
4. **The Workload Pod:** Ephemeral container instance executing application code inside its own network namespace.

> 🎙️ **Speaker Notes:**  
> *"It's crucial to understand the distinction between an Ingress and an Ingress Controller—this is a classic interview question. An Ingress resource is just an API manifest, a set of instructions stored in etcd. On its own, it does absolutely nothing. You need an Ingress Controller running in your cluster. Here, we use the AWS Load Balancer Controller. The controller listens to the Kubernetes API server, detects our voting-ingress manifest, calls the AWS ELBv2 APIs, provisions an internet-facing Application Load Balancer, creates listener rules, and synchronizes target groups with our pod IPs. The Ingress defines the rules; the controller provisions and programs the infrastructure."*

---

## Slide 5 — Architecture Deep Dive
### Dual-Layer Microservice Topology: External Ingress & Internal Mesh

```mermaid
graph TB
    subgraph IngressPerimeter[" 1. External Ingress Boundary (AWS ALB) "]
        Internet(( Internet Traffic ))
        ALB[" AWS Application Load Balancer\n(scheme: internet-facing) "]
        Internet -->|"HTTP :80"| ALB
    end

    subgraph HostRouting[" 2. Layer 7 Routing Logic "]
        RuleVote[" Host: vote.somesh.xyz\nPath: / "]
        RuleResult[" Host: result.somesh.xyz\nPath: / "]
        ALB --> RuleVote
        ALB --> RuleResult
    end

    subgraph K8sCore[" 3. Kubernetes Cluster Internal Network "]
        subgraph Frontends[" Publicly Exposed Services "]
            SvcVote[" Service: voting-service\nType: ClusterIP :80 "]
            SvcResult[" Service: result-service\nType: ClusterIP :80 "]
            RuleVote -.->|"Logical Backend"| SvcVote
            RuleResult -.->|"Logical Backend"| SvcResult
            
            PodVote1[(" Voting Pod 1\n:80 ")]
            PodVote2[(" Voting Pod 2\n:80 ")]
            PodResult[(" Result Pod\n:80 ")]

            SvcVote --- PodVote1
            SvcVote --- PodVote2
            SvcResult --- PodResult
        end

        subgraph PrivateBackend[" Internal-Only Async Pipeline & Data Tier "]
            SvcRedis[" Service: redis\nType: ClusterIP :6379 "]
            PodRedis[(" Redis Pod\n:6379 ")]
            SvcRedis --- PodRedis

            PodWorker[(" Worker Pod\n(No Service - Queue Poller) ")]

            SvcDB[" Service: db\nType: ClusterIP :5432 "]
            PodDB[(" Postgres Pod\n:5432 ")]
            SvcDB --- PodDB
        end
    end

    %% Internal Data Pipeline Links
    PodVote1 ==>|"1. BLPOP / LPUSH"| SvcRedis
    PodVote2 ==>|"1. BLPOP / LPUSH"| SvcRedis
    PodWorker ==>|"2. Polls Tasks"| SvcRedis
    PodWorker ==>|"3. Writes Votes"| SvcDB
    PodResult ==>|"4. SQL Select / Live Feed"| SvcDB

    classDef alb fill:#ff9900,stroke:#232f3e,stroke-width:2px,color:#fff;
    classDef k8s fill:#326ce5,stroke:#232f3e,stroke-width:2px,color:#fff;
    classDef storage fill:#4caf50,stroke:#232f3e,stroke-width:2px,color:#fff;
    class ALB,RuleVote,RuleResult alb;
    class SvcVote,SvcResult,PodVote1,PodVote2,PodResult,PodWorker k8s;
    class SvcRedis,PodRedis,SvcDB,PodDB storage;
```

#### Layer-by-Layer Architectural Separation
* **Edge Ingress Layer:** AWS ALB terminates external client TCP handshakes and inspects HTTP Layer-7 headers.
* **Service Abstraction Layer:** Kubernetes `ClusterIP` Services provide stable internal discovery (`voting-service`, `redis`, `db`, `result-service`) via CoreDNS.
* **Application Execution Layer:** Stateless Python and Node.js pods scale horizontally across cluster worker nodes.
* **Internal Data Pipeline:** Redis and PostgreSQL remain completely invisible to the external network, reachable only via cluster-internal CoreDNS.

> 🎙️ **Speaker Notes:**  
> *"Slide 5 illustrates our complete architecture diagram. Notice the clean segregation between external ingress and internal communications. External users access the cluster exclusively via two hostnames hitting the ALB. Once traffic passes the ALB, notice how internal communications operate: the Voting App connects to redis:6379. It doesn't need hardcoded IP addresses because Kubernetes CoreDNS resolves the Service name redis to its ClusterIP. The Worker pod continuously pulls jobs from Redis and writes records to db:5432. Finally, the Result App queries db to read those counts. PostgreSQL and Redis have zero exposure to the internet, satisfying core defense-in-depth principles."*

---

## Slide 6 — Request Flow
### End-to-End Lifecycle: What Happens When a User Opens `vote.somesh.xyz`?

```mermaid
sequenceDiagram
    autonumber
    actor User as Client Browser
    participant DNS as Route 53 DNS
    participant ALB as AWS Application Load Balancer
    participant TG as ALB Target Group (Pod IPs)
    participant Pod as voting-app-pod (10.0.1.15:80)
    participant Redis as redis-service:6379

    User->>DNS: Query A / CNAME for vote.somesh.xyz
    DNS-->>User: Returns ALB Public DNS / Anycast IPs
    User->>ALB: HTTP GET / (Host: vote.somesh.xyz)
    Note over ALB: Evaluates Ingress Rules<br/>Matches Host: vote.somesh.xyz & Path: /
    ALB->>TG: Selects healthy target (Direct Pod IP)
    Note over TG,Pod: target-type: ip routes directly into<br/>VPC Subnet Pod IP (Bypasses NodePort)
    ALB->>Pod: Forward HTTP Request to 10.0.1.15:80
    Pod->>Pod: Render HTML voting interface
    Pod-->>ALB: HTTP 200 OK (HTML/CSS Payload)
    ALB-->>User: HTTP 200 OK
    User->>ALB: HTTP POST / (Vote: "Cats")
    ALB->>Pod: Forward POST Request
    Pod->>Redis: LPUSH "votes" '{"voter_id":"xyz","vote":"a"}'
    Redis-->>Pod: +OK
    Pod-->>ALB: HTTP 200 (Cookie Set / Vote Acknowledged)
    ALB-->>User: Render "Thank you for voting"
```

#### The Six-Step Request Traversal
1. **DNS Resolution:** Client browser queries `vote.somesh.xyz`. Route 53 resolves the alias record to AWS ALB public Anycast IPs.
2. **Edge Ingestion:** ALB accepts the HTTP connection on port 80 and extracts the HTTP Layer-7 `Host` header.
3. **Ingress Rule Evaluation:** ALB matches `Host == vote.somesh.xyz` and path prefix `/`.
4. **Target Group Selection:** ALB directs request to Target Group `voting-service` (target mode: `ip`).
5. **Direct Pod Forwarding:** ALB routes directly to the private VPC IP of one of the two `voting-app-pod` replicas (`10.0.1.15:80`) with round-robin distribution.
6. **Application Handling:** The Flask container handles the request, interacts with Redis via internal CoreDNS, and returns HTTP 200 back through the ALB to the client.

> 🎙️ **Speaker Notes:**  
> *"Let's trace a packet step-by-step when a user casts a vote. First, the browser resolves vote.somesh.xyz through DNS, which points to the ALB. Second, the ALB terminates the client connection and inspects the HTTP headers. It sees the Host: vote.somesh.xyz header. Third, the ALB matches this against the listener rules generated by our Ingress manifest. Fourth, because our manifest specifies target-type: ip, the ALB does NOT forward to a cluster node's random NodePort. It forwards the request directly to the private VPC IP of one of our voting app pods! Finally, the voting app processes the POST, enqueues the vote onto Redis, and returns the confirmation response. The client experiences minimal latency because the ALB talks directly to the pod."*

---

## Slide 7 — Host-Based Routing
### Single Load Balancer, Multiple Microservices via HTTP Host Headers

```mermaid
flowchart LR
    subgraph ClientRequests[" Inbound Client Requests (Port 80) "]
        ReqA[" HTTP GET /<br/><b>Host: vote.somesh.xyz</b> "]
        ReqB[" HTTP GET /<br/><b>Host: result.somesh.xyz</b> "]
        ReqC[" HTTP GET /<br/><b>Host: api.somesh.xyz</b> "]
    end

    subgraph ALBEngine[" AWS ALB Layer-7 Routing Engine "]
        HeaderParser{" Inspect HTTP 'Host' Header "}
    end

    subgraph RoutingDecision[" Target Group Routing "]
        TG_Vote[" Target Group: voting-app<br/>Targets: Pod 1, Pod 2 "]
        TG_Result[" Target Group: result-app<br/>Targets: Pod 3 "]
        DefaultAction[" Default Action<br/>HTTP 404 / 403 Forbidden "]
    end

    ReqA --> HeaderParser
    ReqB --> HeaderParser
    ReqC --> HeaderParser

    HeaderParser -->|"Matches 'vote.somesh.xyz'"| TG_Vote
    HeaderParser -->|"Matches 'result.somesh.xyz'"| TG_Result
    HeaderParser -->|"Unrecognized Host"| DefaultAction

    style TG_Vote fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    style TG_Result fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style DefaultAction fill:#ffebee,stroke:#d32f2f,stroke-width:2px
```

#### Why Host-Based Routing is Essential for Modern DevOps
* **Virtual Hosting at Scale:** Leverages the standard HTTP/1.1 and HTTP/2 `Host` header to differentiate backend destinations without requiring dedicated public IP addresses.
* **Independent Lifecycle Management:** Voting frontend and result analytics teams can deploy and scale services independently while sharing edge infrastructure.
* **Default Security Fallback:** Inbound scans or unmapped domains hitting the ALB IP directly are dropped immediately at the edge (HTTP 404 / 403).

> 🎙️ **Speaker Notes:**  
> *"Host-based routing is the mechanism that unlocks maximum efficiency in cloud networking. Instead of provisioning separate IP addresses or load balancers for every microservice, the ALB inspects the HTTP Host header in the request. If the header is vote.somesh.xyz, the ALB directs the request to the voting target group. If the header is result.somesh.xyz, it routes to the result dashboard target group. If someone scans the load balancer IP directly or passes an unauthorized domain, the ALB drops the connection. This delivers clean domain branding, high resource efficiency, and centralized traffic management."*

---

## Slide 8 — Understanding the AWS ALB Configuration
### Technical Deep Dive into Manifest Annotations

```yaml
# Source: miniproject-ingress.yaml
metadata:
  name: voting-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listening-port: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
```

```mermaid
flowchart TD
    subgraph NodePortMode[" Legacy Instance Mode (target-type: instance) "]
        ALB1[" AWS ALB "] -->|"Routes to Node IP"| Node1[" EC2 Worker Node\n(NodePort 31234) "]
        Node1 -->|"kube-proxy DNAT / iptables\n(Extra Hop + SNAT Latency)"| Pod1[" Target Pod (Other Node) "]
        style NodePortMode fill:#fff3e0,stroke:#e65100,stroke-width:2px
    end

    subgraph IPMode[" Configured Target Mode (target-type: ip) "]
        ALB2[" AWS ALB "] ===>|"Direct Route via AWS VPC CNI\n(Zero Extra Hops, True Client IP)"| PodDirect[" Target Pod IP: 10.0.2.42:80 "]
        style IPMode fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    end
```

#### Annotation Breakdown & DevOps Implications
1. **`ingressClassName: alb`:** Associates the Ingress with the **AWS Load Balancer Controller** for reconciliation.
2. **`alb.ingress.kubernetes.io/scheme: internet-facing`:** Provisions the AWS ALB in public subnets with public IPs and DNS for public internet access. *(Omitting defaults to `internal`).*
3. **`alb.ingress.kubernetes.io/target-type: ip`:** **Direct Pod Routing.** Targets Pod IP addresses directly via AWS VPC CNI. Eliminates NodePort translation, bypasses `kube-proxy` iptables overhead, and preserves real client IPs.
4. **`alb.ingress.kubernetes.io/listening-port: '[{"HTTP": 80}]'`:** Configures HTTP listener on port 80. *(Note: Canonical controller syntax is `listen-ports`).*

> 🎙️ **Speaker Notes:**  
> *"Slide 8 highlights the exact annotations in our Ingress YAML. First, ingressClassName: alb associates this manifest with the AWS Load Balancer Controller. Second, scheme: internet-facing ensures the ALB gets deployed in public subnets with public DNS names so internet users can vote. Third, and most importantly, target-type: ip. In traditional Kubernetes setups, load balancers route to worker nodes on high-range NodePorts, and kube-proxy routes the packet across nodes using iptables. That introduces extra network hops and hides client IPs. With target-type: ip and AWS VPC CNI, the ALB registers the actual Pod private IPs directly in the ALB Target Group. The load balancer talks directly to the container, cutting latency and eliminating kube-proxy bottlenecks."*

---

## Slide 9 — Ingress vs Service vs Pod
### Architectural Responsibilities & Kubernetes Networking Layers

```mermaid
flowchart TD
    subgraph IngressLayer[" Layer 7: Ingress Resource & ALB "]
        IngressObj[" Ingress / AWS ALB\n- Host & Path Routing\n- SSL Termination Edge\n- HTTP Header Inspection "]
    end

    subgraph ServiceLayer[" Layer 4: Kubernetes Service Abstraction "]
        SvcObj[" voting-service (ClusterIP)\n- Stable Virtual IP (10.100.24.12)\n- Stable Internal DNS Name\n- Dynamic Endpoints Controller Tracking "]
    end

    subgraph WorkloadLayer[" Workload Layer: Pod Replicas "]
        Pod1[" voting-app-pod-1\nIP: 10.0.1.15:80\n(Ephemeral) "]
        Pod2[" voting-app-pod-2\nIP: 10.0.2.42:80\n(Ephemeral) "]
    end

    IngressObj -->|"Defines backend rule for"| SvcObj
    SvcObj -.->|"Selector matches labels"| Pod1
    SvcObj -.->|"Selector matches labels"| Pod2

    style IngressLayer fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style ServiceLayer fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style WorkloadLayer fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
```

#### Layer Responsibilities & Failure Recovery

| Concept | What It Is | Lifecycle & Ephemerality | Why We Need It |
| :--- | :--- | :--- | :--- |
| **Pod** | Container execution instance | **Ephemeral:** Pods die, crash, get rescheduled, and change IP addresses constantly | Executes business logic (Flask, Node, Postgres) |
| **Service** | Stable Layer-4 virtual network endpoint | **Durable:** IP and DNS name remain static for the entire service lifecycle | Solves pod ephemerality by maintaining live endpoints via selectors |
| **Ingress** | Layer-7 entry point & routing rulebook | **Declarative Policy:** Bridges external HTTP traffic to internal Services | Provides host/path routing, SSL termination, and cloud ELB integration |

#### Why the Voting App Has 2 Replicas
* **Fault Tolerance:** `replicas: 2` in `1-voting-app.yaml` ensures that if one pod crashes, the second handles traffic seamlessly.
* **Continuous Endpoint Reconciliation:** The AWS Load Balancer Controller tracks the Service's `EndpointSlices` to keep the ALB Target Group synchronized with healthy pod IPs.

> 🎙️ **Speaker Notes:**  
> *"A frequent interview trap is confusing Ingresses with Services. An Ingress does not replace a Service—they work together. Pods are ephemeral; when a pod restarts, it gets a brand-new IP address. A Kubernetes Service provides a stable, permanent IP and DNS identity for those pods using label selectors. Even with target-type: ip where the ALB routes to pod IPs, the Ingress still references the Service in its YAML. The AWS Load Balancer Controller watches the Kubernetes Endpoints or EndpointSlices managed by the Service to continuously discover and register healthy pod IPs into the AWS Target Group. And notice that our Voting App deployment runs 2 replicas: if Pod 1 terminates, the Service and Target Group immediately reroute all traffic to Pod 2 without user interruption."*

---

## Slide 10 — Internal Application Flow
### Behind the Ingress: Asynchronous Queue & Data Pipeline

```mermaid
flowchart LR
    subgraph Step1[" 1. Ingestion "]
        VotePod[" Voting App Pod\n(Python/Flask) "]
    end

    subgraph Step2[" 2. Buffering "]
        RedisSvc[" redis:6379 "]
        RedisPod[(" Redis In-Memory\nQueue ('votes') ")]
        RedisSvc --- RedisPod
    end

    subgraph Step3[" 3. Processing "]
        WorkerPod[" Worker App Pod\n(.NET Core Daemon) "]
    end

    subgraph Step4[" 4. Storage "]
        DBSvc[" db:5432 "]
        DBPod[(" PostgreSQL 9.4\n(Relational DB) ")]
        DBSvc --- DBPod
    end

    subgraph Step5[" 5. Real-Time Analytics "]
        ResultPod[" Result App Pod\n(Node.js / WebSockets) "]
    end

    VotePod -->|"LPUSH 'votes'"| RedisSvc
    WorkerPod -->|"BRPOP 'votes'"| RedisSvc
    WorkerPod -->|"INSERT INTO votes ..."| DBSvc
    ResultPod -->|"SELECT vote, count(*) ..."| DBSvc
```

#### Why This Asynchronous Pipeline Design Matters
1. **Frontend Protection:** Python frontend pushes a JSON payload onto the Redis `votes` list in microseconds, immediately returning HTTP 200 without waiting for database disk I/O.
2. **Surge Buffering:** Redis absorbs write spikes, protecting PostgreSQL from connection exhaustion.
3. **Decoupled Persistence:** The .NET Worker daemon pulls votes from Redis and executes idempotent SQL statements into PostgreSQL at a stable rate.
4. **Live Visualization:** The Result App queries PostgreSQL (`db:5432`) and streams updated tallies to viewers.

> 🎙️ **Speaker Notes:**  
> *"Slide 10 details what happens inside the cluster after a vote is received. We deliberately decoupled the voting frontend from the database using Redis and a background worker. Imagine an election or television event where 100,000 votes hit per second. If the voting frontend tried to write directly to PostgreSQL, connection pools would saturate and the database would crash. By using Redis as an in-memory queue, the frontend pushes the vote in microseconds and returns a 200 OK. Our worker pod pops votes from Redis at a controlled rate and writes them to PostgreSQL. Then our Result App simply queries PostgreSQL to display the live score. This architecture isolates write spikes and guarantees resilience."*

---

## Slide 11 — Complete End-to-End Flow
### Master Topology: Edge Ingress & Stateful Backend Integration

```mermaid
flowchart TB
    subgraph Clients[" External Traffic Layer "]
        ClientVote[" Voters on Internet "]
        ClientResult[" Viewers on Internet "]
    end

    subgraph AWS_ALB_Boundary[" AWS Application Load Balancer (ALB) "]
        ALB_Listener[" ALB Listener (HTTP :80) "]
        Rule_V[" Rule: Host vote.somesh.xyz "]
        Rule_R[" Rule: Host result.somesh.xyz "]
        TG_V[" Target Group: Voting (IP Mode) "]
        TG_R[" Target Group: Result (IP Mode) "]
        
        ALB_Listener --> Rule_V --> TG_V
        ALB_Listener --> Rule_R --> TG_R
    end

    subgraph EKS_Cluster[" Kubernetes VPC Network Boundary "]
        subgraph Frontends_K8s[" Public Ingress Endpoints "]
            Svc_Vote[" Service: voting-service (ClusterIP) "]
            Svc_Result[" Service: result-service (ClusterIP) "]
            
            Pod_V1[" Voting Pod 1\n(10.0.1.15) "]
            Pod_V2[" Voting Pod 2\n(10.0.2.42) "]
            Pod_R[" Result Pod\n(10.0.2.88) "]
        end

        subgraph Private_K8s[" Private Data & Processing Pipeline "]
            Svc_Redis[" Service: redis (ClusterIP :6379) "]
            Pod_Redis[(" Redis Pod\n(In-Memory Queue) ")]
            
            Pod_Worker[" Worker Pod\n(.NET Daemon) "]
            
            Svc_DB[" Service: db (ClusterIP :5432) "]
            Pod_DB[(" PostgreSQL Pod\n(Port 5432) ")]
        end
    end

    %% External Flow
    ClientVote ==>|"1. vote.somesh.xyz"| ALB_Listener
    ClientResult ==>|"1. result.somesh.xyz"| ALB_Listener
    TG_V ===>|"Direct Route"| Pod_V1
    TG_V ===>|"Direct Route"| Pod_V2
    TG_R ===>|"Direct Route"| Pod_R

    %% Internal Pipeline Flow
    Pod_V1 -->|"2. Enqueue Vote"| Svc_Redis
    Pod_V2 -->|"2. Enqueue Vote"| Svc_Redis
    Svc_Redis --> Pod_Redis
    Pod_Worker -->|"3. Dequeue"| Svc_Redis
    Pod_Worker -->|"4. Persist SQL"| Svc_DB
    Svc_DB --> Pod_DB
    Pod_R -->|"5. Read Tallies"| Svc_DB

    classDef public fill:#ff9900,stroke:#232f3e,stroke-width:2px,color:#fff;
    classDef private fill:#326ce5,stroke:#232f3e,stroke-width:2px,color:#fff;
    classDef storage fill:#00897b,stroke:#232f3e,stroke-width:2px,color:#fff;
    class AWS_ALB_Boundary,ALB_Listener,Rule_V,Rule_R,TG_V,TG_R public;
    class Frontends_K8s,Svc_Vote,Svc_Result,Pod_V1,Pod_V2,Pod_R,Pod_Worker private;
    class Private_K8s,Svc_Redis,Pod_Redis,Svc_DB,Pod_DB storage;
```

> 🎙️ **Speaker Notes:**  
> *"Slide 11 brings our entire system together into one comprehensive diagram. You can clearly trace both paths: On the left, voter traffic hits vote.somesh.xyz, enters the ALB, routes to our voting pods, and streams into the Redis queue. In the center, our worker continuously processes those votes and updates PostgreSQL. On the right, public users viewing the results connect to result.somesh.xyz. Their traffic hits the very same ALB, but the host rule directs them to the Result App, which reads aggregated tallies from PostgreSQL. This unified design gives us centralized security at the cloud boundary, optimal routing via direct pod IPs, and strict network isolation for internal data pipelines."*

---

## Slide 12 — Current Implementation vs YAML Schema Audit
### Truth-in-Engineering: Manifest Inspection & Identified Limitations

```mermaid
flowchart TD
    subgraph ConfiguredState[" What Is Actually Configured in Manifests "]
        C1[" 5 Deployments (Voting:2, Redis:1, Worker:1, Postgres:1, Result:1) "]
        C2[" 4 ClusterIP Services (voting-service, redis, db, result-service) "]
        C3[" Ingress: Host rules for vote.somesh.xyz & result.somesh.xyz "]
        C4[" ALB Annotations: scheme=internet-facing, target-type=ip "]
        style ConfiguredState fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    end

    subgraph IdentifiedIssues[" Identified Manifest Bugs & Schema Limitations "]
        E1[" Ingress Schema: Missing 'http:' wrapper under rules "]
        E2[" Ingress Schema: Lowercase 'prefix' instead of 'Prefix' "]
        E3[" Controller Annotation: 'listening-port' vs 'listen-ports' "]
        E4[" Security: Plaintext DB credentials in ConfigMap/Env "]
        E5[" Persistence: PostgreSQL deployed as ephemeral Deployment (No PVC) "]
        style IdentifiedIssues fill:#ffebee,stroke:#c62828,stroke-width:2px
    end
```

#### Detailed Manifest Audit Findings

| Component / Manifest | Current Configured State | Identified Limitation / Schema Bug | Senior DevOps Assessment |
| :--- | :--- | :--- | :--- |
| **`miniproject-ingress.yaml`** | `networking.k8s.io/v1` Ingress with `ingressClassName: alb` | **Schema Bug:** `paths` defined directly under `host:`, omitting mandatory `http:` block.<br>**Schema Bug:** `pathType: prefix` (lowercase) will fail K8s validation (must be `Prefix`). | `kubectl apply` will fail with validation error. Manifest requires schema correction before deployment. |
| **Ingress Annotations** | `scheme: internet-facing`<br>`target-type: ip`<br>`listening-port: '[{"HTTP": 80}]'` | **Annotation Typo:** AWS Load Balancer Controller canonical key is `listen-ports`, not `listening-port`. | Controller may ignore the port annotation and fallback to defaults or fail to configure listener correctly. |
| **`6-postgres-deploy.yaml`** | `kind: Deployment`, 1 replica, image `postgres:9.4` | **Critical Architecture Risk:** No `PersistentVolumeClaim`. Database storage is ephemeral in container layer. | If the pod restarts, all vote records are irrevocably lost. Version 9.4 reached End-of-Life in 2019. |
| **Database Credentials** | Plaintext `env`: `POSTGRES_USER: postgres`, `POSTGRES_PASSWORD: postgres` | **Security Violation:** Hardcoded plaintext secrets; `POSTGRES_HOST_AUTH_METHOD: trust`. | Credentials exposed in Git and K8s API. Violates compliance standards. |
| **All Workload Deployments** | Basic container specs | **Missing Production Controls:** Zero CPU/Memory `requests` or `limits`; zero `liveness` or `readiness` probes. | Risk of node resource starvation (OOM kills) and routing traffic to unready containers. |

> 🎙️ **Speaker Notes:**  
> *"As a senior DevOps engineer, doing a rigorous manifest audit is mandatory before deploying any code to a cluster. Based strictly on the provided YAML files, I identified several critical issues that must be addressed: First, in miniproject-ingress.yaml, there is a schema violation: the paths key is placed directly under host without the required http wrapper object, and pathType is lowercase prefix instead of title-case Prefix. Kubernetes API validation will reject this manifest immediately upon kubectl apply. Furthermore, the annotation uses listening-port instead of the canonical listen-ports. Second, looking at PostgreSQL: it is configured as a stateless Deployment without a PersistentVolumeClaim. If that pod gets evicted or node dies, all voting data vanishes. In addition, credentials are hardcoded in plaintext and Postgres 9.4 is legacy. Third, none of the deployments define resource requests or health probes. Acknowledging these gaps separates a real DevOps professional from someone blindly copying manifests."*

---

## Slide 13 — Production Improvements
### How I Would Productionize This Architecture

> [!IMPORTANT]
> The features outlined below represent **Recommended Production Improvements** and are not currently configured in the base repository manifests.

```mermaid
flowchart TD
    subgraph EdgeProd[" 1. Edge & Network Hardening "]
        ACM[" AWS Certificate Manager (ACM)\nAutomated TLS 1.3 on Port 443 "]
        WAF[" AWS WAFv2\n(DDoS, Rate Limiting, OWASP Top 10) "]
        R53[" Route 53 Alias Records\n(Failover Routing & Latency Based) "]
    end

    subgraph WorkloadProd[" 2. Workload & Compute Hardening "]
        HPA[" Horizontal Pod Autoscaler (HPA)\nCPU / Custom Metrics Scaling "]
        PDB[" PodDisruptionBudgets (PDB)\nGuaranteed Availability During Drains "]
        Probes[" Liveness & Readiness Probes\nHTTP Health Check Endpoints "]
        ResLimits[" CPU & Memory Requests/Limits\nGuaranteed QoS & Resource Quotas "]
        SecCtx[" SecurityContext: RunAsNonRoot\nRead-only Root Filesystem "]
    end

    subgraph DataProd[" 3. State & Storage Architecture "]
        RDS[" Managed AWS Aurora / RDS PostgreSQL\nMulti-AZ Failover, Automated Backups "]
        ElastiCache[" Managed AWS ElastiCache for Redis\nReplication Groups & Auto-Failover "]
        KMS[" AWS Secrets Manager / SealedSecrets\nEncrypted at Rest with KMS "]
    end

    subgraph ObservabilityProd[" 4. Observability & Telemetry "]
        Prom[" Prometheus & Grafana\nRED Metrics & Node Metrics "]
        OpenTel[" OpenTelemetry Tracing\nJaeger Distributed Traces "]
        FluentBit[" Fluent Bit -> CloudWatch / OpenSearch\nStructured JSON Logging "]
    end
```

#### Production Remediation Checklist
1. **Edge Security & HTTPS:** Bind an AWS Certificate Manager (ACM) TLS 1.3 wildcard certificate (`*.somesh.xyz`) on port 443; add `alb.ingress.kubernetes.io/ssl-redirect: '443'`; attach AWS WAFv2.
2. **State & Database Externalization:** Replace the ephemeral in-cluster PostgreSQL deployment with an **AWS Aurora PostgreSQL Multi-AZ** cluster. Replace Redis with **AWS ElastiCache for Redis**.
3. **Workload Reliability & Autoscaling:** Add `readinessProbe` and `livenessProbe` on HTTP `/`; implement **Horizontal Pod Autoscaling (HPA)** based on CPU utilization and ALB request rates; configure `PodDisruptionBudget`.
4. **Secret Management & Zero Trust:** Move secrets to **AWS Secrets Manager** synced via the **External Secrets Operator (ESO)**; enforce Kubernetes **NetworkPolicies** so only the Worker can connect to PostgreSQL.

> 🎙️ **Speaker Notes:**  
> *"If tasked with taking this project to production for a high-traffic enterprise, here is my implementation roadmap: First, edge security: we must configure HTTPS using AWS Certificate Manager, terminate TLS 1.3 on the ALB, enforce automatic HTTP-to-HTTPS redirects, and attach AWS WAF to mitigate DDoS attacks and vote tampering. Second, data resilience: we should never run a critical database as a standalone pod with ephemeral storage. I would externalize PostgreSQL to an AWS RDS Multi-AZ instance and Redis to AWS ElastiCache. Third, workload resiliency: we need readiness and liveness probes so the ALB never routes traffic to unready pods, resource requests and limits to prevent node starvation, and Horizontal Pod Autoscalers to scale the voting frontend during traffic spikes. Finally, we would introduce Network Policies to restrict pod-to-pod communication, ensuring only the worker can reach the database."*

---

## Slide 14 — DevOps Engineer Takeaway
### Architectural Synthesis & Engineering Principles

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │                        CORE ARCHITECTURAL PILLARS                      │
  ├────────────────────────────────────────────────────────────────────────┤
  │ 1. INGRESS             Centralizes Layer-7 routing policies & domains  │
  │ 2. AWS ALB             Provides an enterprise-grade cloud entry point  │
  │ 3. SERVICES            Delivers stable, durable internal abstraction   │
  │ 4. PODS                Scales stateless application compute elastically│
  │ 5. ASYNC QUEUE         Decouples write surges from storage bottlenecks │
  │ 6. DATA PERSISTENCE    Maintains authoritative state behind the mesh   │
  └────────────────────────────────────────────────────────────────────────┘
```

#### Final Architectural Philosophy
> *"The foundational design principle of production Kubernetes networking is to keep application services internal, private, and decoupled, while using a controlled, declarative Ingress layer to expose only necessary HTTP entry points to the outside world."*

#### Key Achievements in this Architecture
* **Single Load Balancer Economy:** Zero resource waste by serving multiple hostnames through one ALB.
* **Direct Pod Routing:** Eliminating `kube-proxy` bottlenecks via AWS VPC CNI direct IP target groups.
* **Resilient Decoupling:** Complete isolation between synchronous client requests and asynchronous database commits.
* **Production Preparedness:** Clear understanding of current manifest limits and the roadmap required for enterprise reliability.

> 🎙️ **Speaker Notes:**  
> *"To summarize: this project showcases the power of cloud-native Kubernetes architecture when paired with cloud provider primitives. Ingress gives us intelligent Layer-7 host routing; AWS ALB provides an elastic, managed edge; Kubernetes Services provide stable internal discovery; and our asynchronous Redis-Worker pipeline protects backend data integrity. By understanding not just the YAML syntax, but the underlying traffic mechanics, networking trade-offs, and production gaps, we can design scalable, secure, and cost-effective cloud architectures. Thank you, and I am now excited to take any questions."*

---

# Comprehensive DevOps Interview Q&A Guide (20 Questions & Answers)

### Q1: What is Kubernetes Ingress?
**Answer:** Kubernetes Ingress is an API resource (`networking.k8s.io/v1`) that defines application-layer (Layer 7) routing rules for traffic entering a cluster from the outside. It provides capabilities such as host-based routing (virtual hosting), path-based routing, SSL/TLS termination, and centralized traffic management, directing traffic to internal Kubernetes Services.

### Q2: Why did you use Ingress instead of NodePort or LoadBalancer Services?
**Answer:** 
- **Over NodePort:** NodePorts allocate non-standard high ports (30000–32767), require exposing node IPs directly, create security risks, and lack native Layer-7 routing or SSL termination.
- **Over LoadBalancer Services:** Creating a `LoadBalancer` Service for every frontend microservice provisions a separate, costly AWS load balancer for each app. Ingress consolidates traffic for multiple services (`vote.somesh.xyz` and `result.somesh.xyz`) through a single AWS ALB, simplifying DNS management, reducing cloud costs, and centralizing security policies.

### Q3: What is an Ingress Controller, and how does it differ from an Ingress resource?
**Answer:** An Ingress resource is merely a declarative configuration manifest stored in `etcd`. An Ingress Controller is the actual daemon (such as the AWS Load Balancer Controller or NGINX Ingress Controller) running inside the cluster that watches the Kubernetes API for Ingress objects and translates those rules into real routing infrastructure. In this project, the AWS Load Balancer Controller provisions and updates the AWS ALB.

### Q4: What is the role of the AWS ALB in this architecture?
**Answer:** The AWS Application Load Balancer operates at Layer 7 of the OSI model at the edge of the AWS VPC. It acts as the public entry point, accepts inbound client HTTP connections on port 80, evaluates HTTP Host headers, terminates client TCP connections, and forwards traffic directly to container endpoints inside the cluster VPC.

### Q5: Why are ClusterIP Services used behind the Ingress?
**Answer:** `ClusterIP` is the default Kubernetes Service type that allocates an internal-only virtual IP address accessible only from within the cluster. By placing `ClusterIP` services behind the Ingress, backend microservices remain completely shielded from direct internet access. The Ingress/ALB acts as the sole authorized gatekeeper.

### Q6: What is host-based routing, and how does it work here?
**Answer:** Host-based routing uses the HTTP Layer-7 `Host` request header to route requests arriving at the same IP address and port to different backend services. In our ingress configuration, requests with `Host: vote.somesh.xyz` route to `voting-service`, while requests with `Host: result.somesh.xyz` route to `result-service`.

### Q7: What happens when a user accesses `vote.somesh.xyz`?
**Answer:** 
1. The browser resolves `vote.somesh.xyz` to the ALB's IP addresses via Route 53.
2. The browser initiates an HTTP connection to the ALB on port 80.
3. The ALB parses the `Host` header (`vote.somesh.xyz`).
4. The ALB matches the host rule and selects the target group associated with `voting-service`.
5. The ALB forwards the request directly to one of the two `voting-app` Pod IPs (`target-type: ip`).
6. The pod returns the rendered HTML page back through the ALB to the client.

### Q8: How does traffic reach the Pod with `target-type: ip`?
**Answer:** In AWS EKS using the AWS VPC CNI plugin, every Pod is assigned a real, routable IP address from the VPC subnet. When `alb.ingress.kubernetes.io/target-type: ip` is specified, the AWS Load Balancer Controller registers the Pod IPs directly into the ALB Target Group. The ALB routes traffic directly to the container, completely bypassing worker node NodePorts and `kube-proxy` iptables DNAT routing.

### Q9: What does `alb.ingress.kubernetes.io/scheme: internet-facing` mean?
**Answer:** It tells the AWS Load Balancer Controller to deploy the ALB in public subnets with an internet-routable public IP and a publicly resolvable DNS name. If omitted or set to `internal`, the ALB would be provisioned in private subnets, accessible only within the VPC or over a corporate VPN/DirectConnect.

### Q10: What happens if one of the Voting App Pods fails?
**Answer:** Because `1-voting-app.yaml` defines `replicas: 2`, the deployment controller continuously reconciles the desired state and starts a replacement pod. Concurrently, the ALB target group health checks fail on the dead pod, causing the ALB to immediately stop sending traffic to that IP and forward 100% of requests to the surviving healthy replica with zero downtime.

### Q11: Why are there two replicas for the Voting App but only one for the Result App?
**Answer:** In the current manifests, the Voting App has `replicas: 2` to handle higher concurrent write loads and demonstrate high availability for the primary user flow. The Result App has `replicas: 1` as a baseline single-instance deployment. In production, the Result App should also have at least 2 replicas alongside Horizontal Pod Autoscaling (HPA).

### Q12: Why is Redis used between the Voting App and PostgreSQL?
**Answer:** Redis acts as an in-memory queue and buffer. Ingesting votes into a relational database synchronously under high traffic leads to connection exhaustion, lock contention, and high latency. Pushing votes into Redis takes fractions of a millisecond, decoupling fast frontend ingestion from database persistence.

### Q13: What is the role of the Worker deployment?
**Answer:** The Worker is an asynchronous consumer daemon written in .NET. It continuously polls or blocks on the Redis `votes` queue, extracts raw voting events, processes the payload, and writes the vote record into PostgreSQL. Notice that the Worker does not have a Kubernetes Service because no other component sends inbound network traffic to it.

### Q14: Why does PostgreSQL have a ClusterIP Service named `db`?
**Answer:** Both the Worker and the Result App need to connect to PostgreSQL over TCP port 5432. Pod IPs are ephemeral and change whenever a pod restarts. The ClusterIP Service `db` gives PostgreSQL a stable internal DNS name (`db.default.svc.cluster.local`) and virtual IP that never changes.

### Q15: Did you spot any schema bugs or syntax errors in the provided Ingress manifest?
**Answer:** Yes, three key issues:
1. **Schema Violation:** Under `spec.rules[].host`, the manifest places `paths:` directly without the required `http:` parent key required by `networking.k8s.io/v1`.
2. **Invalid Capitalization:** `pathType: prefix` is lowercase; the Kubernetes v1 schema requires `Prefix`, `Exact`, or `ImplementationSpecific`.
3. **Annotation Typo:** The annotation is written as `alb.ingress.kubernetes.io/listening-port`, whereas the official AWS Load Balancer Controller annotation is `alb.ingress.kubernetes.io/listen-ports`.

### Q16: How would you enable HTTPS/TLS in this architecture?
**Answer:**
1. Request a public wildcard SSL certificate (`*.somesh.xyz`) in AWS Certificate Manager (ACM).
2. Add annotations to the Ingress:
   ```yaml
   alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
   alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/xxx
   alb.ingress.kubernetes.io/ssl-redirect: '443'
   ```
3. The ALB will terminate TLS at the edge and automatically redirect HTTP port 80 traffic to HTTPS port 443.

### Q17: How would you make PostgreSQL production-ready?
**Answer:** In production, I would not run PostgreSQL as a single-replica Deployment with ephemeral storage. I would externalize it to **AWS Aurora PostgreSQL** or **Amazon RDS Multi-AZ**. If required to run in-cluster, I would deploy it as a `StatefulSet` attached to persistent EBS volumes via `PersistentVolumeClaims` (using the AWS EBS CSI driver) with automated snapshotting and replication.

### Q18: How would you secure the database credentials currently in plaintext?
**Answer:** Currently, `POSTGRES_USER` and `POSTGRES_PASSWORD` are hardcoded in plaintext in `6-postgres-deploy.yaml`. In production, I would store credentials in **AWS Secrets Manager**, encrypt them with AWS KMS, and synchronize them into Kubernetes Secrets using the **External Secrets Operator (ESO)** or CSI Secrets Store Driver, referencing them in the pod via `secretKeyRef`.

### Q19: What would you add for pod health checking and observability?
**Answer:** 
- **Health Probes:** Add `readinessProbe` and `livenessProbe` to all deployments so Kubernetes and the ALB do not route traffic to uninitialized or deadlocked pods.
- **Resource Limits:** Define `resources.requests` and `resources.limits` for CPU and Memory to prevent noisy neighbors and out-of-memory node crashes.
- **Observability:** Deploy Prometheus and Grafana for metrics, Fluent Bit for shipping container logs to Amazon CloudWatch / OpenSearch, and OpenTelemetry for distributed request tracing.

### Q20: How would you implement network isolation inside the cluster?
**Answer:** By default, Kubernetes has a flat network where all pods can communicate with all other pods. In production, I would apply **Kubernetes NetworkPolicies** enforcing zero-trust:
- Allow ingress traffic only to `voting-app` and `result-app` pods.
- Allow access to `redis` only from `voting-app` and `worker`.
- Allow access to `db` only from `worker` and `result-app`.
- Block all direct internet egress from the database pod.

---

### Corrected Production Ingress Manifest
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: voting-ingress
  namespace: default
  labels:
    app.kubernetes.io/name: voting-app-ingress
    app: voting-app
  annotations:
    # AWS Load Balancer Controller Settings
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    # SSL & Security Hardening (Production Ready)
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    # alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/example-uuid
    # Health Check Customization
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
spec:
  ingressClassName: alb
  rules:
    - host: vote.somesh.xyz
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: voting-service
                port:
                  number: 80
    - host: result.somesh.xyz
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: result-service
                port:
                  number: 80
```
