# theATL.social Backend Infrastructure

## __Update: This doc is a bit out-of-date as of 12/23! Will update as soon as I have an opportunity.__

## About

This document is a combination of my notes taken while setting up theATL.social, and the general outline of how I established the server. The intended reader of this document are those:

- Interested to set up their own Mastodon sites and are looking for examples
- Those interested in joining `theATL.social`, but wanting to make sure the server is not going to disappear on a whim
- Any future sysops of `theATL.social`

## General Infrastructure Notes

- `theATL.social`'s infrastucture operates using containers for Mastodon microservices, operating via `docker compose`-based daemons on droplets and object storage in DigitalOcean
- Logging is captured via the Docker `awslogs` plugin and piped and analyzed to AWS CloudWatch

## Network Setup
- I need to redact the juicy bits from public view, however...all but the forward proxy sits behind a firewall. Ports are secured with least access needed, ACLs are set, and all known best practices are implemented.

## Instances and Microservices

- Unless otherwise stated, each instance runs the respective container(s) in daemon mode.

### Mastodon API + Streamer
- API and Streamer operate as separate containers on same (computing) instance, orchestrated via `docker compose`

#### Scaling Plan
- Horizontal scaling

#### Mastodon API
- [Ruby on Rails](https://rubyonrails.org)-based service
- REST API Interface
- HTTPS endpoint runs [Puma](https://puma.io)
- Stateless
- GETs messages via the ActivityPub spec to the Fediverse
- Messages vis the ActivityPub spec are POSTed to the API from the Fediverse

#### Mastodon Streamer
- NodeJS-based service
- Provides notifications to mobile devices and connected web-clients
- Connects via HTTP long pull or WebSockets


### Queue Processor

#### Scaling Plan
- Horizontal Scaling

#### Notes
- [Sidekiq](https://github.com/mperham/sidekiq): Ruby-based service
- Queues write-type operations, including
  - Writes to PostgreSQL
  - Transaction emails
  - Other write operations
- Stateless, pulls from Redis queue

### PostgreSQL Database Instances

#### Instances
- Operated on 2 Instances:
  - Primary PostgreSQL Read/Write Databse
  - Read-Only Replica with WAL Logs streamed from Primary database

#### Backups and Resiliency
- Automated `pgdump` backups every 30 minutes
- Hot standby PostgreSQL read replica to read-write if needed

#### Scaling Plan
- Vertical Scaling

#### Notes
- The "permanent record" of a Mastodon service's state
- No special configuration needed other than pgTune
- Primarily vertical scaling; horizontal scaling through use of read replicas in a hot standby confiburaiton

### Redis

#### Instances
- Operating on one instance in stand-alone configuation

#### Scaling Plan
- Vertical at first
- Would transition to managed cluster instance if vertical scaling is not sufficient

#### About
- Maintains the queue of Sidekiq jobs
- Also feeds and volatile cache
- Horizontally scalable through a Redis cluster configuration
- Memory-only

#### Object Storage

#### Instances
- Operated on managed service

#### Scaling
- Hypothetically, none needed

#### About
- Cached copies of posts from other servers
- Posts from own server
- Stores all media
- Tendency to get a bit large, as any servers' interaction with other servers will result in a data copy/transfer

### ElasticSearch

#### Status
- Funds not available to operate at the moment
- Would utilize managed service (AWS OpenSearch)

#### About
- For text-based searches of all cached post data
- Expensive to operate
- Basic text search used in lieu of ElasticSearch

### Nginx Reverse Proxy

#### Status
- Operational for HTTPs endpoint

#### Scaling Plan
- Vertical, then Horizontal scaling if needed
- Would be an endpoint to which haproxy forward proxy would load balance

#### About
- Sits on top of the HTTPS endpoint
- Relays requests to:
  - Static compiled React pages
  - REST API
  - Streamer/WebSockets


### HAProxy Forward Proxy

#### Status
- Operational

#### Scaling Plan
- Vertical
- If needed, horizontal scaling with DNS-based load balancing

#### About
- (Final) HTTPs endpoint
- Load balancer to Nginx


## References Utilized

- https://softwaremill.com/the-architecture-of-mastodon/
