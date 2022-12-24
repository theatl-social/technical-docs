# Utilizing Cloudflare's R2 Service as Object Store for Mastodon Server

## Motivation

Self-hosted Mastodon servers must establish a location for media files to be cached. Media files include avatars and other user-uploaded content to the self-hosted server - and content collected by Sidekiq in the process of an /inbox `POST`. 

_(Sidekiq is the queue processor for Mastodon. The processor receives requests and data from other servers, and processes those requests as resources permit. Requests may include posts containing images or video. Default settings on Mastodon direct Sidekiq to download a copy of the media and store that copy locally, for access by server users)_

By default, Mastodon will store media files on the instance's local drive. 

This storage method, however, is typically ill-advised because local or block storage is typically more expensive than object storage. And, while local/block storage may be expanded, this step typically requires some type of user intervention. Mastodon media stores also tend to be a bit large depending on the number of users and their activities: assumptions of 100GB to 1T are not unreasonable.

- _Block storage refers to storage such as Amazon Web Service's (AWS) Elastic Block store (EBS) or DigitalOcean's Volumns._
- _Object storage refers to storage such as AWS S3, DigitalOcean Spaces, or Cloudflare R2_

When I evaluating the options for Block Storage, I chose [Cloudflare's R2 Service](https://www.cloudflare.com/products/r2/) due to its pricing model of free egress, low cost on read operations, and reasonable cost on write operations. More importantly, however, Cloudflare easily allows the serving of an object store via a custom domain on their platform. The data served is distributed to a CDN and cached appropriately, which in turn further lowers costs.

## Objective

- Utilize R2 as an object store in the Mastodon Stack

## Prerequistes

1. Self-Hosted Mastodon Instance (not a hosted instance)
2. CloudFlare Account (Free account is OK) with R2 service activated (requires billing method)
3. [AWS CLI](https://aws.amazon.com/cli/])
4. Optional - Existing local storage or block store from which data will be transferred into R2
5. rclone - Utility to copy from local storage (or AWS S3 or DigitalOcean, etc.) to R2

Note: These steps have only been tested on a Linux/MacOS computer, and not Windows.

## Steps

### Step 1: Create an R2 Bucket
1. On your account page, go to the `R2` section, and then click "Create Bucket"
2. Choose a descriptive name for your bucket, and click "Create Bucket"
3. You have now created your object store! Now, return to the `R2` section on the main Cloudflare page.
4. Create R2 API Tokens, then click "Create API Token". Add `Edit` _and_ `Write` Permissions. Copy the `Access Token` and `Token Secret` into a secure location.
5. Connect the bucket to a custom domain [Cloudflare Documentation Instructions](https://developers.cloudflare.com/r2/data-access/public-buckets/#custom-domains-configuration)
6. Make sure you selected the option to create a DNS entry.
7. __Recommended__: Upload a file manually via the Cloudflare UI, and then see if you can acess the file via your custom domain, using your web browser. If you can access the file, you're almost done!
8. [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)! The reason why the AWS CLI is a prerequisite is because we are going to use that service to set our CORS setting on our R2 bucket.
9. Go to a terminal/command line. At the command line, presuming that the `aws` command from the AWS CLI is now in your `PATH`, type: `aws configure`. You will be prompted for the `Access Key` and `Secret` that you received from enabling the Cloudflare R2 API. For region and other values, use the default values.
10. Copy the file [`recommended_r2_cors.json`](https://github.com/theatl-social/technical-docs/blob/main/recommended_cors.json) from this repo into your working folder. (Review this JSON file first - you may want to limit the origin to your primary Mastodon server domain. If not, the file will work as-is)
11. Collect your Cloudfront Account ID. This ID is located on your `R2` page in the Cloudfront Control Pnael.
12. We will now be using the AWS CLI to set the CORS setting for the R2 bucket. At the command line, input the following (replacing the values in the brackets with the approproiate values).
```
aws s3api put-bucket-cors --bucket <BUCKET NAME> --endpoint-url https://<CLOUDFLARE ACCOUNT ID>.r2.cloudflarestorage.com --cors-configuration file://<PATH TO WORKING DIRECTORY WITH LEADING SLASH>/recommended_r2_cors.json
```
13. If the command succeeds, you have now sucessfully set the CORS options for the bucket.

## Step 2: Configure Mastodon to Utilize the Bucket

Mastodon settings are stored in environment variables, or `dotenv` files, or through other means. The following instructions are agnostic as to how you are setting your environment variables.

1. Input the following Mastodon variables into your Mastodon Stack, replacing the instructions in the brackets with the respective values:
```
S3_ENABLED=true
S3_BUCKET=<YOUR BUCKET NAME -- NOT YOUR CUSTOM DOMAIN NAME>
S3_ENDPOINT=https://<YOUR CLOUDFLARE ACCOUNT ID>.r2.cloudflarestorage.com
S3_ALIAS_HOST=<YOUR CUSTOM DOMAIN NAME>
S3_PERMISSION=private
AWS_ACCESS_KEY_ID=<YOUR ACCESS KEY>
AWS_SECRET_ACCESS_KEY=<YOUR SECRET ACCESS KEY>
```

Now, restart your instance, and everything should now work OK! 

## Optional Step 3: Copy files over from an existing object store

__Important__: The folder/directory structure must be precisely the same on the destination and origin object stores. 

1. Make sure you have `rclone` installed. If you are using Homebrew, just type `brew install rclone`. If you are on a Linux distro without Homebrew, `rclone` may be available via a package manager.
2. The [following instructions](https://forum.rclone.org/t/cloudflare-r2-now-working-with-rclone/30730) will show you how to configure `rclone` to access Cloudflare R2. Note that the instructions refer a beta version of `rclone`, however, the current stable version has this capacity.
3. Repeat similar steps, depending on your source object store, to set up configuration on your source.
4. Utilize `rclone sync <src> <dest>` to copy your files from one object store to another. Hint: use `--transfer` and `--checkers` to multi-thread the copy and transfer threads of the process. Documentation is here: `https://rclone.org/docs/`

## Done!
