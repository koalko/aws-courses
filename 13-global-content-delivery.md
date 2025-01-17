# Global content delivery and optimization

## Cloudfront Architecture

`Origin` - the source location of your content. Can be `S3 Origin` or `Custom Origin`.

`Distribution` – the configuration unit of CloudFront.

`Edge Location` – local cache of your data.

`Regional Edge Cache` – larger version of an edge location. Provides another layer of caching.

When user tries to access content cached by CloudFront, the order to check is:

1. Edge Location
2. Regional Edge Cache
3. Origin Fetch (cache miss)

CloudFront integrates with ACM for HTTPS.

There is no caching for uploads (upload directly to origin).

CloudFront `Origins` are linked to `Behaviours`, which in turn are linked to `Distributions`. CloudFront Distribution always have at least one Behaviour, but it also can have more. Default behaviour is wildcard. More specific behaviours take precedence. Origins, Origin Groups, TTL, Protocol Policies, restricted access are configured via Behaviours.

## CloudFront Behaviours

Distribution allow to configure:

- price class: which edge locations content is distributed into (all / subsets)
- WAF (Web Application Firewall)
- alternate domain names
- SSL certificate (default/custom)
- security policy (TLS version; balance between security and accessibility)
- supported HTTP versions

Behaviour allow to configure:

- path pattern
- origin
- objects compression
- viewer protocol policy (HTTP/HTTPS options)
- allowed HTTP methods
- viewer access restriction (via trusted key groups / trusted signer; requires signed URLs or signed cookies)
- cache policy
- function associations

## TTL and Invalidations

Default TTL (behaviour) is 24 hours (validity period). You can set minimum TTL and maximum TTL (limit per-object TTL settings).

Important origin headers:

- `Cache-Control: max-age=&lt;seconds&gt;` (per-object TTL)
- `Cache-Control: s-maxage=&lt;seconds&gt;` (per-object TTL)
- `Expires: <day-name>, <day> <month> <year> <hour>:<minute>:<second> GMT` (per-object expiration date/time)

These headers can be set using custom origin or S3 (via object metadata).

Cache invalidation is performed on a distribution and applies to all edge locations. It takes time and expires every object (within a distribution) which fits a path pattern (path may contain asterisk).

An alternative to invalidation is to upload the file with an altered name, and make the application point to a new file. Common pattern is to use versioned file names: `top_kek_v1.jpg`, `top_kek_v2.jpg`, `top_kek_v3.jpg`, etc.

## AWS Certificate Manager (ACM)

ACM lets you run a public or a private Certificate Authority (CA). `Private CA`: applications need to trust your CA. `Public CA`: browsers trust a list of providers, which can trust other providers. ACM can generate or import certificates. If generated – it can **automatically renew**. If imported – **you are responsible for renewal**. Certificates can be deployed out to **supported services** (CloudFront, ALBs are supported, while EC2 **is not**).

ACM is a regional service. Certificates cannot leave the region they are generated or imported in. To use a certificate in a regional service (for example with ALB in `ap-southeast-2`), you need a certificate in ACM in the same region (`ap-southeast-2` in this example). Global services (such as CloudFront) operate as though within `us-east-1`.

## CloudFront and SSL

Each CloudFront distribution receives a default domain name (CNAME record), for example: `https://d111111abcdef8.cloudfront.net`. SSL is supported by default for `*.cloudfront.net` domains. Also, an alternate domain names can be used. To use HTTPS with alternate domains you need to generate or import a matching certificate in `ACM` (in `us-east-1`).

Possible values for a `viewer protocol policy` option:

- HTTP or HTTPS
- HTTP -> HTTPS
- HTTPS only

Two SSL connections, both need **valid public** certificates:

1. Viewer -> CloudFront
2. CloudFront -> Origin

Historically every SSL-enabled site needed its own IP. Encryption starts at the TCP connection; and `Host` header happens after that (`Layer 7 – Application`).

In 2003 a `SNI` (Server Name Indication) extension was added to TLS to allow a host to be included: during TLS handshake a client can tell a server which hostname to access. Old browsers don't support SNI; and CloudFront charges extra ($600/month/distribution) for dedicated IP.

## Origin Types & Origin Architecture

Possible origin types:

- Amazon S3 Bucket
  - Can have different origin access types:
    - public
    - origin access control settings
    - legacy access identities (OAI)
  - You can also add custom headers
- AWS Media Package Channel Endpoint
- AWS Media Store Container Endpoint
- Custom origin (web server, S3 static website)
  - Can specify options:
    - Path
    - Viwer protocol policy
      - HTTP only
      - HTTPS only
      - Match viewer
    - Custom HTTP/HTTPS ports
    - Minimum origin SSL protocol
    - Custom headers

## Adding a CDN to a static website

1. Create an S3 bucket with public access (including proper bucket policy) and static website hosting enabled.
2. Upload the website' files to S3 bucket.
3. Create a CloudFront distribution: select S3 bucket as an origin and set a default root object to an index file (`index.html`, for example).

## Invalidating a CloudFront distribution

Go to distribution -> invalidations and create invalidation with desired paths.

## Adding a Route 53 domain to CloudFront distribution

Go to distribution, edit its settings, add a custom domain as an alternate domain name (CNAME). Request a public certificate for a domain name. Open the newly created certificate and press `Create record in Route 53` to proceed with DNS check. Select a newly created certificate in a distribution settings. Wait for CloudFront distribution deployment to complete. Create an A-record for a Route 53 hosted zone. Select `Alias for CloudFront distribution` for `Route traffic to` and select a distribution.

## Securing CF and S3 using OAI

Origin Access Identity (OAI) is a type of identity, which can be associated with CloudFront distributions. CloudFront "becomes" specific OAI, which can be used in S3 bucket policies. Common pattern is to deny all, and allow one or more OAI(s).

Custom origin can be secured using:

- custom headers
- firewall around origin which only allows CloudFront Edge Locations IPs traffic

## Private Distribution & Behaviours

With private distribution behaviour every request require signed cookie or URL. You can have multiple behaviours, each public or private, and it can allow to redirect unauthorized access to a private behaviour to a public one.

To make a behaviour private you need to assign a signer to it.

Old way to create a private behaviour:

- Account Root User creates a CloudFront Key
- the account is added as a **trusted signer**

New (preferred) way to create a private behaviour: create a trusted key groups and assign them as signer.

Signed URL provides access to **one object**. _Historically_ RTMP distributions couldn't use cookies. Use Signed URLs if your client doesn't support cookies.

Cookies provides access to groups of objects. It is also good to use cookies if maintaining application URL is important.

Example of a private distribution behaviour setup:

- S3 origin configured with OAI
- public behaviour with following settings:
  - `Path: Default(*)`
  - `Origin: APIGW`
  - `Forward Cookies: All`
  - `Restrict Viewer Access: No`
- private behaviour with following settings:
  - `Path: images/`
  - `Origin: S3`
  - `Forward Cookies: No`
  - `Restrict Viewer Access: Yes`
- client requests the access to images
- Lambda signer behind the API GW generates the cookie using trusted key groups and sends the cookie to client
- client requests the images, sending the cookie along

## Securing S3 bucket via OAI

1. Go to the CloudFront origin settings.
2. Enable `Origin access control settings` and create a control setting.
3. Copy policy, go to S3 bucket permissions and paste the copied policy.
4. Save the changes and wait until distribution deployment has finished.

## Lambda@Edge

Allows to run lightweight Lambda at edge locations. These functions can adjust data between the viewer and origin. Currently only Node.js and Python runtime is supported. Run in the AWS Public Space (not VPC). Lambda layers are not supported. Lambda@Edge also has different limits vs normal Lambda functions.

Four types of Lambda@Edge (basically hooks for a particular phase):

- `Viewer Request`: client -> edge location
- `Viewer Response`: edge location -> client
- `Origin Request`: edge location -> origin
- `Origin Response`: origin -> edge location

Limits:

- `Viewer`: `128 MB` memory, `5 seconds` timeout
- `Origin`: Normal Lambda memory, `30 seconds` timeout

Some usage examples:

- A/B testing
- Migration between S3 origins
- Different objects based on device
- Content by country

## Global Accelerator

Starts with 2x `anycast` IP Addresses: `1.2.3.4` and `4.3.2.1`. Anycast IPs allow a single IP to be in multiple locations. Routing moves traffic to closest location. Traffic initially uses public internet and enters a Global Accelerator edge location. From the edge, data transits globally across the AWS global backbone network. Less hops, directly under AWS control, significantly better performance.

Summary:

- Moves the AWS network closer to customers
- Connections enter at edge using anycast IPs
- Transit over AWS backbone to 1+ locations
- Can be used for non-HTTP(S) (`TCP`/`UDP`) (_difference from CloudFront_)

## Quiz

- CloudFront is a ... (choose one)
  - A Global CDN capable of caching static & dynamic content
- What features are used together to ensure S3 buckets can only be accessed via CloudFront
  - OAI
  - Bucket Policies
- What region must certificates be created in when using ACM
  - the region the service which uses them is in
- To use an ACM cert with CloudFront what region must it be created in
  - us-east-1
- Which of the following are services ACM supports
  - CloudFront
  - Api Gateway
- You have a TCP based application which is used globally. You want to improve the network performance for global users. Which service might support this requirement?
  - Global Accelerator
