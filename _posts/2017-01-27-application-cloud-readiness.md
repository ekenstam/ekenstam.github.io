---
layout: post
title: "Migrating to AWS, Part 1 — Application Cloud Readiness"
thumbnail: /assets/cloud/aws-cloud-icon.png
published: true
categories: ['Software Development','Cloud Computing']
tags: cloud aws microservices encryption resiliancy
---

If you are writing a new web service or application that will be hosted in the cloud you can design and implement it to be 'cloud native' from the beginning. But what about legacy services/applications that have been designed to run in an on-premises datacenter? Several common changes will need to be made to most services and applications in preparation for moving to the cloud. While some of these may be application specific, this post discusses several changes for application cloud readiness that will be broadly applicable to most applications and can be implemented while the application is still hosted in the on-premises datacenter.

The primary issues that need to be addressed as part of application cloud readiness include:

* Automated Regression Tests
* On-board to API Gateway
* Service/Application Decoupling
* Data Encryption
* Remove use of other on-premises services/dependencies
* Implement Application Resiliency

I am currently working on migrating applications from Intuit's on-premises datacenters to the Amazon Web Services (AWS) cloud computing platform and this post contains some AWS examples. However, most of the issues discussed would be applicable to other cloud providers as well.

## Automated Regression Tests

Most of the functionality impacted by the changes described in this post should already be covered by existing regression tests, but where there are gaps in regression test coverage it is critically important to add automated tests to fill those gaps as part of the effort to migrate the application to the cloud. It should go without saying that maintaining the quality of the existing service/application functionality is critical while refactoring for cloud readiness.

## On-board to API Gateway

Many companies expose their web services (microservices) to the Internet through an API Gateway to provide common functionality (caching, throttling, etc.), monitoring and security to all services. If your organization is using an API Gateway (or Service Gateway), the first step is to make sure your service is on-boarded to that gateway, if applicable. Depending on the API Gateway being used it may be possible for the service to be exposed via the API Gateway while the service is still hosted on-premises.

If the application is not a web service (e.g. a batch process, etc.) then on-boarding to the API Gateway may not be necessary. If your service will not use an API Gateway then you may need to implement additional functionality, such as throttling and security, in the service itself.

[![AWS API Gateway Diagram](/assets/cloud/aws-RequestProcessingWorkflow.jpg)](https://aws.amazon.com/api-gateway/details/)

### Implement API Gateway Authorization/Enforcement

After the service is on-boarded to the API Gateway, all clients of the service must be migrated to use the API Gateway endpoint (i.e. don't call the service GTM endpoint directly). The API Gateway will send requests to the service with an authorization token which the application will validate. The application must reject all requests that don't have a valid authorization token.

**Validation**

* Positive 'Happy Path' tests through API Gateway endpoint with correct 'Authorization' token. Expect 200 OK.
* Negative tests through API Gateway endpoint with incorrect 'Authorization' token. Expect 403 Forbidden.
* Negative tests through direct GTM endpoint with correct 'Authorization' token. Expect 403 Forbidden.

### On-board as Client of Dependencies in API Gateway

Your service may depend on other services, in which case all requests from your service to other services should be sent through the API Gateway. In addition to being a best practice, this will make migration to the cloud easier since you will only need to insure your cloud environment has network access to the API Gateway, and not a potentially large number of direct server endpoints which may change as those services are also migrated to the cloud.

For each service dependency, onboard as a client of that service in the API Gateway. Use the client credentials and API Gateway endpoint of the service to send all requests. All applications may only access services they depend on via the API Gateway.

**Validation**

* Inspect service/application properties and config files for non-API Gateway URLs.

## Service/Application Decoupling

Unless the application is a monolith and you are following a 'lift and shift' cloud migration strategy, the service or application should be refactored and decoupled from other services/applications at a reasonable granularity prior to cloud migration. This generally means implementing a 'Microservices Architecture'.

There are many aspects of a Microservices Architecture and reasonable people may differ on exactly what a Microservices Architecture is. However, the main issue discussed in this post is that all shared access to application database tables must be removed; all tables should only have one application reading and writing to them. I believe if this one guideline is adhered to most other aspects of Microservices Architecture will follow.


### Migrate Tables to Private Database

The service/application must only update a ‘private’ database. No other service or application may update the ‘private’ database. Private database tables may be replicated to read-only replicas, if needed. 

* Move all application database tables to a private database/schema
* Expose web services to share data you own with other applications/services
* Create read-only replicated tables where neither of the above two options are feasible

> [Microservices](https://www.martinfowler.com/articles/microservices.html) prefer letting each service manage its own database, either different instances of the same database technology, or entirely different database systems - an approach called [Polyglot Persistence](https://www.martinfowler.com/bliki/PolyglotPersistence.html).
> &mdash; <cite>Martin Fowler<cite>
> [![Microservice Decentralized Data Diagram](https://www.martinfowler.com/articles/microservices/images/decentralised-data.png)](https://www.martinfowler.com/articles/microservices.html)

Aside from being a desirable architecture, Microservices Architecture is important for cloud migration since each service may be deployed in it's own cloud environment (e.g. separate AWS environments and/or VPCs) to increase security and [reduce blast radius](https://aws.amazon.com/answers/account-management/aws-multi-account-security-strategy/).

**Validation**

* Inspect service/application properties and config files for 'non-private' database connection strings.
* Verify permissions of DB objects are restricted to only the service/application and authorized users.

### Convert Access to Non-private Databases to Web Service Calls

The service must not update any databases it does not 'own'. Updating objects owned by other services is done via web service calls.

* Call web services (instead of directly reading another service's private database) to retrieve/update data you don't own

## Data Encryption

Your organization may have specific requirements for protecting data in the cloud using [encryption](https://www.owasp.org/index.php/Guide_to_Cryptography). One level of protection is 'transparent data encryption' or disk encryption which should be enabled for all cloud resources as a best practice, except if all the data is considered 'public' (press releases, advertisements, etc.). For certain types of data (e.g. date of birth, social security number, financial account numbers, etc.), application-level data encryption may also be required. Regardless of your data encryption method, you will need to [safely and securely manage the encryption keys](https://www.sans.org/security-resources/policies/general/pdf/acceptable-encryption-policy) for your application.

> “When you boil it down, transparent encryption with associated internal key management is a very simple and cost effective way to secure data at rest, but not effective for securing keys from database administrators or determined hackers who gain access to the host.”
> &mdash; <cite>[Securosis, “Understanding and Selecting a Database Encryption or Tokenization Solution”, 2010](https://securosis.com/assets/library/reports/Securosis_Understanding_DBEncryption.V_.1_.pdf)<cite>

### Migrate to 'Cloud Friendly' Key Management System

Your legacy service/application may manage 'secrets' (encryption keys, passwords, etc.) in a manner that is not conducive to the cloud. Refactor your application so that it uses a key management system that is available both on-premises and in the cloud. Make sure you fully understand the nuances of key management and apply best practices.

**Validation**

* Inspect service/application properties and config files and insure there is no on-premises specific connection information or 'in the clear' secrets.

### Perform Data Classification of all Database Columns

If you haven't already, it is recommended that you classify all database columns per your organization's policy. For example:

* Public (e.g. press releases, advertisements)
* Restricted (e.g. production logs)
* Sensitive (e.g. name, phone number, email)
* Highly Sensitive (e.g. date of birth, social security number, financial account numbers)
* Secret (e.g. cryptographic secrets, passwords)

The classification of each column in the database should be added to the comments of the column using the `COMMENT ON COLUMN` SQL command.

**Validation**

* Query the `COMMENT` for all columns in all tables of the private database and insure the value is not `NULL` and contains expected values.

### Column-level Encryption of Required Database Columns

Based on the column's data classification, your organization's policy may require additional column-level encryption for highly sensitive data. For all columns of a certain data classification or higher or based on industry regulatory requirements (PCS, etc.), implement column-level application or transparent data encryption, as required. Note that this is not strictly an issue of migrating to the cloud, but a security best practice.

**Validation**

* For each column identified as 'Highly Sensitive' or 'Secret', inspect the values stored in those columns and insure they are not in clear text.

### Implement tokenization for searching of encrypted columns

When encrypting data, a [salt](https://en.wikipedia.org/wiki/Salt_(cryptography)) may be used to further protect the data from attacks. This results in the same data having different encrypted values. If there is a requirement to search encrypted values, an otherwise meaningless static token of the unencrypted value ([tokenization](https://en.wikipedia.org/wiki/Tokenization_(data_security))) must be used.

**Validation**

* Add automated tests for all functionality that allow searching, querying or grouping on encrypted columns.

## Remove use of other on-premises services/dependencies

OK, admittedly this is a catch-all requirement and encompasses much of the above items. There may be other application-specific resources that are only available in the on-premises datacenter. A cloud-friendly replacement must be found for all such resources. For example, you cannot use local filesystems or NFS mounts for permanent storage in the cloud. While the application is still running in the on-premises datacenter, you may need to refactor file storage to use cloud storage such as AWS S3 or Glacier (and make those accessible from the on-premises datacenter). Other refactoring may need to be done on a case-by-case basis depending on the needs of the particular application/service.

**Validation**

* Inspect service/application properties and config files for any directory or filename paths or other types of connection or location configuration.

## Implement Application Resiliency

Application resiliency is important regardless of where the application is deployed; on-premises or the cloud. However, the [fundamentals of cloud service reliability](https://blogs.microsoft.com/microsoftsecure/2012/09/12/fundamentals-of-cloud-service-reliability/) is such that failures of specific cloud resources will eventually occur. Just moving a system into the cloud doesn’t make it fault-tolerant or highly available.

Whereas in your on-premises datacenter you may achieve reliability by purchasing 'enterprise-class' hardware with redundant components, advanced monitoring capabilities and a dedicated IT staff, the cloud largely uses commodity hardware and achieves reliability by providing access to redundant resources across availability zones. Reliability in the cloud is based on the architecture of the application being able to take full advantage of these redundant resources that the cloud provides.

[![AWS High Availability Diagram](/assets/cloud/aws-high-availability.jpg)](http://media.amazonwebservices.com/architecturecenter/AWS_ac_ra_ftha_04.pdf)

The following tools and techniques should be applied as appropriate based on the needs of the application.

* [Hystrix](https://github.com/Netflix/Hystrix)
* Retry mechanism (perhaps adaptive or exponential backoff retry)
* Queuing
* Caching
* Degraded behavior

**Validation**

* Identify dependencies not wrapped with [Hystrix](https://github.com/Netflix/Hystrix).
* Automated resiliency tests performing [Blue/Green deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html) of the service/application while under load.
* Automated resiliency tests using [Destructive Proxy](https://github.com/intuit/destructive_socks5_proxy) and/or [WireMock](http://wiremock.org/).
* Automated tests using [Chaos Monkey](https://github.com/netflix/chaosmonkey), Simian Army or similar tools.

## Copyright

<a rel="license" href="http://creativecommons.org/licenses/by-nc-nd/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-nd/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-nd/4.0/">Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License</a>.

<!-- Link references -->
