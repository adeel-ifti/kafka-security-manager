[![Build Status](https://travis-ci.org/simplesteph/kafka-security-manager.svg?branch=master)](https://travis-ci.org/simplesteph/kafka-security-manager)

# Kafka Security Manager

Kafka Security Manager (KSM) allows you to manage your Kafka ACLs at scale by leveraging an external source as the source of truth. Zookeeper just contains a copy of the ACLs instead of being the source.

![Kafka Security Manager Diagram](https://i.imgur.com/BuikeuB.png)

There are several advantages to this:
- **Kafka administration is done outside of Kafka:** anyone with access to the external ACL source can manage Kafka Security
- **Prevents intruders:** if someone were to add ACLs to Kafka using the CLI, they would be reverted by KSM within 10 seconds. 
- **Full auditability:** KSM provides the guarantee that ACLs in Kafka are those in the external source. Additionally, if for example your external source is GitHub, then PRs, PR approvals and commit history will provide Audit the full log of who did what to the ACLs and when
- **Notifications**: KSM can notify external channels (such as Slack) in order to give feedback to admins when ACLs are changed. This is particularly useful to ensure that 1) ACL changes are correctly applied 2) ACL are not changed in Kafka directly.

Your role is to ensure that Kafka Security Manager is never down, as it is now a custodian of your ACL.

A sample CSV to manage ACL is:
```
KafkaPrincipal,ResourceType,ResourceName,Operation,PermissionType,Host
User:alice,Topic,foo,Read,Allow,*
User:bob,Group,bar,Write,Deny,12.34.56.78
User:peter,Cluster,kafka-cluster,Create,Allow,*
``` 

# Building

``` 
sbt clean test
sbt universal:stage
```

This is a Scala app and therefore should run on the JVM like any other application

# Artifacts

By using the JAR dependency, you can create your own `SourceAcl`.

SNAPSHOTS artifacts are deployed to [Sonatype](https://oss.sonatype.org/content/repositories/snapshots/com/github/simplesteph/ksm/)

RELEASES artifacts are deployed to [Maven Central](http://search.maven.org/#search%7Cga%7C1%7Ccom.github.simplesteph):

`build.sbt` (see [Maven Central](http://search.maven.org/#search%7Cga%7C1%7Ccom.github.simplesteph) for the latest `version`)
```
libraryDependencies += "com.github.simplesteph" %% "kafka-security-manager" % "version"
```

# Configuration 

## Security configuration

Make sure the app is using a property file and launch options similar to your broker so that it can 
1. Authenticate to Zookeeper using secure credentials (usually done with JAAS)
2. Apply Zookeeper ACL if enabled

*Kafka Security Manager does not connect to Kafka.* 

Sample run for a typical SASL Setup:
``` 
target/universal/stage/bin/kafka-security-manager -Djava.security.auth.login.config=conf/jaas.conf
```

Where `conf/jaas.conf` contains something like:
```
Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/kafka/secrets/zkclient1.keytab"
    principal="zkclient/example.com@EXAMPLE.COM";
};
```

## Configuration file

For a list of configuration see [application.conf](src/main/resources/application.conf). You can customise them using environment variables or create your own `application.conf` file and pass it at runtime doing:
```
target/universal/stage/bin/kafka-security-manager -Dconfig.file=path/to/config-file.conf
```

Overall we use the [lightbend config](https://github.com/lightbend/config) library to configure this project.

## Environment variables 
The [default configurations](src/main/resources/application.conf) can be overwritten using the following environment variables:

- `KSM_READONLY=false`: enables KSM to synchronize from an External ACL source. The default value is `true`, which prevents KSM from altering ACLs in Zookeeper
- `KSM_EXTRACT=true`: enable extract mode (get all the ACLs from Kafka formatted as a CSV)
- `KSM_REFRESH_FREQUENCY_MS=10000`: how often to check for changes in ACLs in Kafka and in the Source. 10000 ms by default
- `AUTHORIZER_CLASS`: override the authorizer class if you're not using the `SimpleAclAuthorizer`
- `AUTHORIZER_ZOOKEEPER_CONNECT`: zookeeper connection string
- `AUTHORIZER_ZOOKEEPER_SET_ACL=true` (default `false`): set to true if you want your ACLs in Zookeeper to be secure (you probably do want them to be secure) - when in doubt set as the same as your Kafka brokers.  
- `SOURCE_CLASS`: Source class. Valid values include
    - `com.github.simplesteph.ksm.source.NoSourceAcl` (default): No source for the ACLs. Only use with `KSM_READONLY=true`
    - `com.github.simplesteph.ksm.source.FileSourceAcl`: get the ACL source from a file on disk. Good for POC
    - `com.github.simplesteph.ksm.source.GitHubSourceAcl`: get the ACL from GitHub. Great to get started quickly and store the ACL securely under version control.
- `NOTIFICATION_CLASS`: Class for notification in case of ACL changes in Kafka. 
    - `com.github.simplesteph.ksm.notification.ConsoleNotification` (default): Print changes to the console. Useful for logging
    - `com.github.simplesteph.ksm.notification.SlackNotification`: Send notifications to a Slack channel (useful for devops / admin team)

# Running on Docker

## Building the image

```
./build-docker.sh
```

## Docker Hub

Alternatively, you can get the automatically built Docker images on [Docker Hub](https://hub.docker.com/r/simplesteph/kafka-security-manager)  

## Running

(read above for configuration details)

Then apply to the docker run using for example (in EXTRACT mode):

```
docker run -it -e AUTHORIZER_ZOOKEEPER_CONNECT="zookeeper-url:2181" -e EXTRACT=true \
            simplesteph/kafka-security-manager:latest
```

Any of the environment variables described above can be used by the docker run command with the `-e ` options. 

## Example

```
docker-compose up -d
docker-compose logs kafka-security-manager
# view the logs, have fun changing example/acls.csv
docker-compose down
```

For full usage of the docker-compose file see [kafka-stack-docker-compose](https://github.com/simplesteph/kafka-stack-docker-compose)

Add the entry to your `/etc/hosts` file
```
127.0.0.1 kafka1
```

## Extracting ACLs

You can initially extract all your existing ACL in Kafka by running the program with the config `extract=true` or `export EXTRACT=true`

Output should look like:
```
[2018-03-06 21:49:44,704] INFO Running ACL Extraction mode (ExtractAcl)
[2018-03-06 21:49:44,704] INFO Getting ACLs from Kafka (ExtractAcl)
[2018-03-06 21:49:44,704] INFO Closing Authorizer (ExtractAcl)

KafkaPrincipal,ResourceType,ResourceName,Operation,PermissionType,Host
User:bob,Group,bar,Write,Deny,12.34.56.78
User:alice,Topic,foo,Read,Allow,*
User:peter,Cluster,kafka-cluster,Create,Allow,*
```

You can then use place this CSV anywhere and use it as your source of truth. 

# External API
To activate this feature, set `FEATURE_GRPC=true`.

## gRPC Endpoint
Kafka Security Manager exposes a GRPC endpoint to be consumed by external systems. The use case is to build a UI on top of KSM or some level of automation. 

By default KSM binds to port 50051, but you can configure this using the `GRPC_PORT` environment variable.

## gRPC Gateway (REST Endpoint)
By default KSM binds to port 50052, but you can configure this using the `GRPC_GATEWAY_PORT` environment variable.

This provides a REST API to consume data from KSM. Swagger definition is provided at [src/main/resources/specs/KsmService.yml](src/main/resources/specs/KsmService.yml)

## Service Definition

The API is defined according to the proto file in [src/main/protobuf/](src/main/protobuf/)

# Compatibility

KSM Version | Kafka Version
--- | ---
0.4-SNAPSHOT | 1.1.x
0.3 | 1.1.x
0.2 | 1.1.x (upgrade to 0.3 recommended)
0.1 | 1.0.x (might work for earlier versions)

# Contributing

You can break the API / configs as long as we haven't reached 1.0. Each API break would introduce a new version number. 

PRs are welcome, especially with the following:
- Code refactoring  / cleanup / renaming
- External Sources for ACLs (JDBC, Microsoft AD, etc...)
- Notification Channels (Email, etc...) 

Please open an issue before opening a PR.



Mohammed Anis Ur Rahman 
Position: GCP Platform Engineering Lead
Date: 29/03/2019

Category 
Category 	comments	Ratings
Previous platform engineer / Devops / infrastructure experience 	- 3 years of AWS experience with focus on AWS Route 53, Load balancer, regions, cloud formation, Terraform and Ansbile
- No particular GCP experience
- 1 year of Azure experience 
	3
Questions asked around AWS Infrastructure, architecture, networking and security and other similar previous work experience. 	- Some correct answers around AWS EC2 instances, regions and infrastructure. 
- Lack of technical dept in questions around key management, IAM roles, security, networking	3
Questions around Terraform (questions around TF State, TF Plan vs TF init, what are modules used for? TF providers and provisioners and differences) 	30% correct answers around basic Terraform and state. Rest of the answers lack technical depth and hands on experience	2
Kubernetes ( what’s a kubernetes Pod vs Node). What is namespaces, what’s RBAC? Kubernetes Services. Scaling clusters? How does the pipeline work for kubernetes? 	40% correct answers only. Good experience around Jenkins, maven, ant. 
Basic experience of Kubernetes and k8s eco system (Istio, networking, security).	2
Questions around Docker exec, docker build, docker compose. Docker Swarm. Communication skills, ability to explain technical concepts in depth?	Basic understanding around Docker. 	2
Cloud migration experience, end to end migration strategy, security and compliance related experience?	Mainly worked as Devops and wasn’t directly involved with migration or lift and shift style projects	1
CI / CD pipelines, creating pipelines, managing environments, Jenkins, how do we extend Jenkins? 	Only basic knowledge (30% correct answers)	2
Questions around Infrastructure Bootstrap. Platform AWS: IAM rules? Examples IAM roles and groups. Authentication and Authorisation with AD. Users and groups. 	Good knowledge around AWS infrastructure. Basic knowledge around IAM roles, users and groups and AD authentication or similar technologies	2
Understanding of monitoring and alerting tools, previous experiences around ELK Stack, prometheus, nagios, filebeat etc?	Very basic understanding. Lacks practical experience.	1
Ansible (and other configuration management systems), playbook, tasks and roles, what’s the difference?	Mostly incorrect answers. 	1



