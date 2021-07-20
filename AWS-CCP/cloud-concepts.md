# Domain 1: Cloud Concepts

**Topics**
-   [1.1 Define the AWS Cloud and its value proposition](#11-define-the-aws-cloud-and-its-value-proposition)
-   [1.2 Identify aspects of AWS Cloud economics](#12-identify-aspects-of-aws-cloud-economics)
-   [1.3 Explain the different cloud architecture design principles](#13-explain-the-different-cloud-architecture-design-principles)

---
## 1.1 Define the AWS Cloud and its value proposition
### Define the benefits of the AWS cloud including:
-   **Security**
    -   AWS is responsible for managing ifrastructure security, which offloads this responsibility from their customers.
-   **Reliability**
    -   AWS provides pretty high uptime/availability/durability guarancies for their services.
-   **High Availability**
    -   AWS provides very easy way of how to prepare high available applications, basic HA setups are as simple as introducint a ELB (Elastic Load Balancer) and multiAZ (Availability Zone) deployments.
-   **Elasticity**
    -   Elasticity is one of the biggest features of any public cloud provider. It allows customer to "tweak" resources of given services based on their needs. Upscale resources during peak hours, downscale or even shutdown during off-peak hours - while paying only for the real usage.
-   **Agility**
    -   Building resources in AWS is very agile - EC2 instances are buildable within a minutes just with a simple button click. This allows application teams to be very agile when it comes to deployment to different environments.   
-   **Pay-as-you go pricing**
    -   With AWS, customer has the option to pay in a as you go model, which means there are no upfront payments if customer does not want them. However upfront payments are usually tied with some sort of discounts.
-   **Scalability**
    -   Closely tied to Elasticity attribute - there are numerous ways i AWS how to scale your workloads. Manually or even automatically based on some events / metrics. This makes the AWS a lot more scalable when compared to on-premises datacenters.
-   **Global Reach**
    -   AWS provides services across the world in different geo regions. There is no problem to run workloads in US, EU, ASIA, AFRICA - customer can choose the region closest to their customers, or fulfill some regulations.
-   **Economy of scale**
    -   AWS has built a vast infrastructure and can therefore benefit from lower prices coming from the scale. Customers using AWS can therefore benefit from the same - buying the same infrastructure alone would cost number of times more.

### Explain how the AWS cloud allows users to focus on business value
-   **Shifting technical resources to revenue-generating activities as opposed to managing infrastructure**
    -   When using AWS, customer can off-load number of activities to AWS and focus on the more important parts. Managing infrastructure is perfect example - when runnning onpremises, there are high costs of managing own hardware and having specalized teams with such knowledge. In AWS, one can simply buy a managed service, or virtual machine without the need to manage the lower layers.

---
## 1.2 Identify aspects of AWS Cloud economics
### Define items that would be part of a Total Cost of Ownership proposal
-   **Understand the role of operational expenses (OpEx)**
    -   Operational expenses is the part usually tied with public clouds - these are ongoing expenses. Company switching to public cloud would have 0 CapEx coming from their own infrastructure, while having 100% OpEx.
-   **Understand the role of capital expenses (CapEx)**
    -   Capital expenses are usually counted as the expenses to buy own infrastruture. Company running on onpremises usually has these expenses higher than companies running in cloud.
-   **Understand labor costs associated with on-premises operations**
    -   Runnin onpremises operations brings significant labor costs as well -> company has to hire & pay teams to support and maintain the infrastructure. When migrating to cloud, such teams can be significantly reduced and company can focus more on the app-development side.
-   **Understand the impact of software licensing costs when moving to the cloud**
    -   Running onpremises operations brings also licensing cost as well. Many companies running onpremises are using some virtualization platforms, which counts to a licensing cost which can be totally ignored when running in cloud.

### Identify which operations will reduce costs by moving to the cloud
-   **Right-sized infrastructure**
    -   Pay only for what you really use -> use small resources off-peak, scale up during peaks.
-   **Benefits of automation**
    -   Auto scale resources based on need -> reduce costs when we dont need big resource buffer.
    -   Auto-shutdown resources when not needed.
-   **Reduce compliance scope (for example, reporting)**
    -   Leveraging AWS resources will ensure compliance with several certifications. Since AWS manages the infrastructure begind, they are responsible for some compliance parts, thus offloading some from customer.
-   **Managed services (for example, RDS, ECS, EKS, DynamoDB)**
    -   Having managed resources can save engineering costs in company -> why having a K8s admin team if you can buy the service directly -> EKS

---
## 1.3 Explain the different cloud architecture design principles
### Explain the design principles
-   **Design for failure**
    -   In human words - if architecture is prepared with belief that something will fail, it is then better prepared for the actual fail, and chances to survive catastrophic situations increases.
-   **Decouple components versus monolithic architecture**
    -   In human words - having microservices loosely coupled helps increase the availability of a whole system. If one microservice fails - others will survive. In extreme situations, one microservice does not even has to know about a failure of another.
-   **Implement elasticity in the cloud versus on-premises**
    -   In human words - cloud is basicaly a synonymous to the word elasticity. A lot of services in cloud can be very elastic in terms of increasing the resources, scaling up/down, different kind of automations and conditional behaviour. Having the same level of elasticity in on-premises world is very very hard to achieve (if not impossible).
-   **Think parallel**
    -   In human words - try to parallelize & automate everything you can. Having 1mil of requests spread across 1000 servers will result in far better performance of your application than having only 1 server drowned with all requests. Cloud environment supports this approach and it is generally best practice to design applications to leverage this.
