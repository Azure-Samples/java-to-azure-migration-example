# Java to Azure Migration Guideline
This repos list all the documents migration your java application to azure

# Migration guidelines
## Messaging
### Service Bus
#### RabbitMQ to Service Bus
1)  Spring AMQP
2)  Spring JMS
3)  Spring Stream API?
4)  Java CLient

#### IBM MQ to Service Bus
1)  Spring JMS

### Event Hub
#### Kafka
1) Enable Kafka support for Event Hub

## Storage
### Azure Storage Blob
#### S3 to Azure Storage Blob
1) Spring S3 to Spring Azure Storage Blob

### Azure File
1) Mount your file to ACA

## No-SQL DB
### CosmosDB
1)  Enable Cassadra support for CosmosDB
1)  Enable Mongo Support to CosmosDB

## Cache
### Azure Redis Cache
1) Spring Cache for Redis

## Hard-Coded Secrets and Connection Strings 
### Use KeyVault to manage your secret
1) Spring Boot
### Enable MI to connect your resource
#### SQL DB
1) Spring Boot
2) Java
#### No SQL DB
1) Spring Boot
2) Java

## In Process Session
### Use Redis to manage the session
1) [Use Redis for Spring Session](./Session/Use-Redis-for-Spring-Session.md)

## Java App Authentication 
### AAD
1) Migration from LDAP to AAD
