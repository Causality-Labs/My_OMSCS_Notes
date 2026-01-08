
## Internet Technologies

- Internet services expose functionality via web interfaces, with users issuing browser-based HTTP requests.
- Typical architecture uses three tiers: presentation (static layout/UI), business logic (dynamic, user-specific processing), and database (storage/management).
- These tiers can co-reside in one process/machine (e.g., Apache handling both presentation and logic) or be split across multiple components.
- Middleware coordinates interactions among tiers; multiprocess deployments rely on IPC mechanisms to exchange data.

## Internet Service Architectures

- High or bursty request rates often demand multi-process, multi-node deployments.
- Scaling out adds more nodes running the service, fronted by a load balancer that distributes incoming requests.
- Behind the balancer:
    - **Homogeneous** cluster: each node can handle any request end-to-end.
    - **Heterogeneous** cluster: nodes specialize in certain pipeline stages or request types.

## Homogenous Architecture

- Functional homogeneity: every node can handle any request end-to-end, regardless of type.
- Load balancer stays simple—just distribute requests (e.g., round-robin) without tracking capabilities per node.
- Data may be replicated or distributed, but each node has access to whatever it needs to serve the request.
- Downsides: limited ability to exploit caching/locality because the front-end doesn’t track which node holds warm data.


## Cloud Computing Poster Child: Animoto

- Amazon’s retail workload was highly seasonal, leaving much of its infrastructure idle outside peak holiday demand.
- In 2006 Amazon exposed its excess capacity via web APIs, launching **Amazon Web Services** and **EC2**, enabling third parties to rent compute resources.
- Animoto, a photo-to-video service, initially ran on ~50 EC2 instances instead of building its own datacenter.
- After integrating with Facebook in 2008, Animoto saw explosive growth—750k new users in three days—and scaled from 50 to 3,400 EC2 instances within a week.
- Such rapid, two-orders-of-magnitude scaling would have been impossible with traditional owned hardware, demonstrating cloud elasticity’s value.


## Cloud Computing Requirements

- Traditional provisioning bought enough hardware for peak demand; any surge beyond capacity meant dropped requests and lost business.
- ![](https://assets.omscs.io/notes/57336766-5450-410E-A7EA-44AD6A68CEEC.png)
- Ideal cloud behavior: resource capacity tracks demand instantly so costs scale with actual usage.
- ![](https://assets.omscs.io/notes/17DAC6A2-5849-4D9C-83AC-4A5CE3E1FD76.png)
- Goals of cloud computing: on-demand elastic services, pay-per-use pricing, professionally managed infrastructure, and API-driven access—even if the customer doesn’t own the hardware.


## Cloud Computing Overview

- Cloud computing exposes shared resources—compute, storage, networking, and higher-level services (e.g., email, databases)—through APIs.
- Access/control APIs can be web, library, or CLI based, reached over the Internet.
- Providers offer varied pricing (spot, reservation, etc.); billing usually uses coarse-grained tiers (e.g., instance sizes) rather than per-cycle usage.
- Cloud vendors manage the infrastructure; common management stacks include OpenStack and VMware vSphere

## Why does Cloud Computing Work

- **Law of large numbers:** aggregate demand across many clients averages out, so overall resource usage stays relatively stable even if individual workloads spike—letting a cloud provider meet variable peaks with a fixed resource pool.
- **Economies of scale:** hosting many customers on shared hardware amortizes infrastructure costs per user, making large-scale cloud deployments cost-efficient.

## Cloud Computing Vision

- Cloud vision: computing delivered like a public utility—hardware becomes fungible and location-agnostic.
- Virtualization underpins this abstraction but can’t hide every hardware dependency.
- Provider-specific APIs introduce portability/lock-in issues; switching clouds often requires code changes.
- Entrusting core business code to external providers raises privacy and security concerns.


## Cloud Deployment Models

- **Public cloud:** provider owns the infrastructure; third-party tenants rent compute/storage/network resources on demand.
- **Private cloud:** one organization owns/operates the hardware and software, using cloud tech for flexible, in-house elasticity.
- **Hybrid cloud:** blends private and public resources—core workloads run privately while overflow, failover, or auxiliary tasks (e.g., load testing) burst to the public cloud.
- **Community cloud:** a public cloud shared by a specific group with common requirements.


## Cloud Service Models

- Cloud service models range from fully managed applications to raw infrastructure.
- ![](https://assets.omscs.io/notes/81DBAA16-6AF2-4CEB-9840-665F49F5001B.png)
- **SaaS:** provider delivers complete apps (e.g., Gmail); customers just use the service.
- **PaaS:** provider supplies an execution environment and APIs (e.g., Google App Engine) so developers can build/run apps without managing OS/tooling.
- **IaaS:** provider offers virtualized compute, storage, and networking (e.g., Amazon EC2); customers manage the OS and above, often sharing physical hardware with other tenants (though dedicated instances exist).


## Requirements for the Cloud

- Cloud resources must be **fungible**—easily repurposed to meet varied customer needs—otherwise the economic model breaks.
- Resource management must dynamically adjust allocations based on demand and operate at massive scale (tens of thousands of nodes).
- Large-scale systems inevitably face failures; clouds need built-in fault-handling mechanisms.
- Multi-tenant environments demand strong performance isolation so one tenant can’t impact others or the provider.
- Providers must ensure data safety and secure execution environments for all clients.


## Cloud Enabling Technologies

- Virtualization makes resources fungible so they can be repurposed across customers.
- Provisioning and scheduling stacks (e.g., Mesos, YARN) spin up resources quickly and consistently.
- Big-data engines like Hadoop MapReduce and Spark handle large-scale processing/storage needs.
- Storage layer relies on distributed, often append-only filesystems, plus NoSQL databases and in-memory caches for scalable data access.
- Isolation tech enforces per-tenant resource slices for security/performance.
- Monitoring/telemetry tools (Flume, CloudWatch, Log Insight) give providers and customers visibility into infrastructure and application behavior.


## The Cloud as a Big Data Engine

- Cloud computing gives anyone “infinite” resources—if you can pay, you can tackle data- and compute-intensive problems.
- Big data services require at least storage and processing layers, often supplemented with fast caching for repeated access.
- Developer-friendly interfaces (SQL, Python, etc.) sit atop these stacks so teams can query data in familiar languages.
- Analytics, mining, and machine learning toolkits (algorithms, apps, visualizations) are typically bundled.
- Because data often streams in continuously, platforms must support ingestion and staging of real-time feeds.

## Example Big Data Stacks

### Hadoop

  

![](https://assets.omscs.io/notes/690C46E9-31A8-4841-BA34-660234AE758A.png)

  

### Berkeley Data Analytics Stack (BDAS)

  

![](https://assets.omscs.io/notes/686C631F-E544-4048-AF40-A21BB31E7427.png)

