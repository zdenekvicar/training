# Domain 2: Security and Compliance

**Topics**
-   [2.1 Define the AWS shared responsibility model](#21-define-the-aws-shared-responsibility-model)
-   [2.2 Define AWS Cloud security and compliance concepts](#22-define-aws-cloud-security-and-compliance-concepts)
-   [2.3 Identify AWS access management capabilities](#23-identify-aws-access-management-capabilities)
-   [2.4 Identify resources for security support](#24-identify-resources-for-security-support)

---
## 2.1 Define the AWS shared responsibility model
### Recognize the elements of the Shared Responsibility Model
-   Security and Compliance are working in a shared responsibility model in AWS. Dividing responsibilities into:
    -   Security in the Cloud -> Being customers responsibility
    -   Security of the Cloud -> Being AWS responsibility

### Describe the customer’s responsibly on AWS
-   Areas where customer is responsible for:
    -   Customer data
    -   Platform, Applications, Identity & Access management
    -   Operating System, Network & Firewall configuration
        -   Client-side data, encryption & data integrity, authentication
        -   Server-side encryption (File system and/or data)
        -   Networking traffic, protection(encryption, integrity, identity)
- Describe how the customer’s responsibilities may shift depending on the service used (for example with RDS, Lambda, or EC2)
    -   EC2 -> Customer is responsible for complete maintenance of OS and above (security patches, encryption of volumes, ...)
    -   Lambda -> Customer is responsible for the code itself (its security, ...)
    -   RDS -> Customer is mainly (among other things) responsible for the data (encryption, ...)

### Describe AWS responsibilities
-   Areas where AWS is responsible for:
    -   Software
        -   Compute
        -   Storage
        -   Database
        -   Networking
    -   HW / AWS Global Infrastructure
        -   Regions
        -   Availability Zones
        -   Edge Locations

---
## 2.2 Define AWS Cloud security and compliance concepts
### Identify where to find AWS compliance information
-   Locations of lists of recognized available compliance controls (for example, HIPPA,SOCs)
    -   https://aws.amazon.com/compliance/programs/ provides a list of all compliance programs AWS offers, including mentioned HIPPA, SOC [1-3], PCI-DSS and others
-   Recognize that compliance requirements vary among AWS services
    -   https://aws.amazon.com/compliance/services-in-scope/ provides a list of all services which are in compliance with various programs
    -   Not every service supports every program -> if customer heavily relies (regulations) on some programs like PCI-DSS, selection of appropriate services should be driven by this list
-   At a high level, describe how customers achieve compliance on AWS
    -   Choosing the right services which supports required compliance program
    -   Choosing features of given service that meets the requirements for such compliance program
    -   Every AWS service which handles client data provides both `Encryption at rest` and `Encryption in transit` ways by using AWS KSM or AWS CloudHSM with AES-256
-   Describe who enables encryption on AWS for a given service
    -   Encryption has to be enabled by customer, as it falls under customer's responsibility. Enablement ifself has to be done from an account/user with sufficient privileges.
-   Recognize there are services that will aid in auditing and reporting
        o Recognize that logs exist for auditing and monitoring (do not have to understand the logs)
    -   AWS supports various services which helps customers with Auditing, Reporting and Monitoring.
    -   [Amazon CloudWatch](https://aws.amazon.com/cloudwatch) -> Observability of your AWS resources and applications on AWS and on-premises
    -   [AWS Config](https://aws.amazon.com/config) -> Record and evaluate configurations of your AWS resources
    -   [AWS CloudTrail](https://aws.amazon.com/cloudtrail) -> Track user activity and API usage
-   Explain the concept of least privileged access
    -   Best practices in Cloud security says, everyone should have the minimum possible privileges required to perform his job.
        -   Example: Developer who needs to read data from S3 bucket, should have readonly access without an option to delete data and/or bucket.
        -   Example: Security person who investigate audit trail data, should not have access to EC2 or S3 instances as he does not need to create/control these for his job.

---
## 2.3 Identify AWS access management capabilities
### Understand the purpose of User and Identity Management
-   AWS IAM helps with managing access for different users to different AWS services at one place
-   It helps with or supports:
    -   Setting up of account wide password policies, like complexity, enforced rotation, etc.
    -   Setting and/or enforcing MFA authentication for AWS users
    -   Provides various resources to work with:
        -   Users to create separate AWS accounts for each employee
        -   Groups to configure shared permissions to multiple users
        -   Roles to delegate access to users/services that normally dont have such access
        -   Policies to define and control the permissions
            -   Managed policies -> created & managed by AWS, comes with some pre-defined permissions
            -   Custom policies -> created & managed by customer, can be customized to meet the specific needs of each customer
-   Root account is the master of the universe when it comes to AWS account and as such, there are some best practices when it comes to this account
    -   Root account should not be normally used for daily work -> create user with enough privileges instead
    -   Root account should have MFA enabled
    -   Root access keys should not be used
    -   Tasks which only root account is privileged to perform:
        -   AWS account changes
            -   Account name
            -   Email address
            -   Root user password
            -   Root user access keys
        -   Restore IAM user permissions
        -   Activate IAM access to Billing and Cost Management console
        -   Certain tax invoices are Root only
        -   Close AWS account
        -   Change AWS support plan
        -   Register a seller in Reserved Instance Marketplace
        -   Configure MFA on S3 bucket
        -   Edit/Delete S3 bucket policy with invalid VPC ID or VPC Endpoint ID
        -   Sign up for GovCloud

---
## 2.4 Identify resources for security support
### Recognize there are different network security capabilities
-   Customer using AWS cloud can utilize different services and products to help with network security
    -   Native AWS services like Security Groups, Network ACLs, AWS WAF
    -   Find 3rd party product on AWS Marketplace

### Recognize there is documentation and where to find it (for example, best practices, whitepapers, official documents)
-   AWS Knowledge Center -> https://aws.amazon.com/premiumsupport/knowledge-center/
-   AWS Security Center -> https://aws.amazon.com/security/
-   AWS Security forum -> https://forums.aws.amazon.com/category.jspa?categoryID=27 
-   AWS Security blogs -> https://aws.amazon.com/blogs/security/
-   AWS whitepapers -> https://aws.amazon.com/whitepapers/

### Know that security checks are a component of AWS Trusted Advisor
-   AWS Trusted Advisor analyzes your AWS env and provides best practices tips on following areas:
    -   Cost optimization
    -   Performance
    -   Security
    -   Fault tolerance
    -   Service quotas
-   NOTE: Some of the checks are not available for Free-tier support plans
