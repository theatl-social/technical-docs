# Utilizing Cloudflare's R2 Service as Object Store for Mastodon Server

## Motivation

Self-hosted Mastodon servers must establish a location for media files to be cached. Media files include avatars and other user-uploaded content to the self-hosted server - and content collected by Sidekiq in the process of an /inbox `POST`. 

_(Sidekiq is the queue processor for Mastodon. The processor receives requests and data from other servers, and processes those requests as resources permit. Requests may include posts containing images or video. Default settings on Mastodon direct Sidekiq to download a copy of the media and store that copy locally, for access by server users)_

By default, Mastodon will store media files on the instance's local drive. 

This storage method, however, is typically ill-advised because local or block storage is typically more expensive than object storage. And, while local/block storage may be expanded, this step typically requires some type of user intervention. Mastodon media stores also tend to be a bit large depending on the number of users and their activities: assumptions of 100GB to 1T are not unreasonable.

- _Block storage refers to storage such as Amazon Web Service's (AWS) Elastic Block store (EBS) or DigitalOcean's Volumns._
- _Object storage refers to storage such as AWS S3, DigitalOcean Spaces, or Cloudflare R2_

When I evaluating the options for Block Storage, I chose (Cloudflare's R2 Service)[https://www.cloudflare.com/products/r2/] due to its pricing model of free egress, low cost on read operations, and reasonable cost on write operations. More importantly, however, Cloudflare easily allows the serving of an object store via a custom domain on their platform. The data served is distributed to a CDN and cached appropriately, which in turn further lowers costs.

