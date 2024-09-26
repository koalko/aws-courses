# Route 53 - Global DNS

R53 is a globally resilient service (multiple DNS servers).

## Hosted Zones

R53 Hosted Zone is a DNS DB for a domain. Created with domain registration via R53; also can be created separately. Zone hosts DNS records (A, AAAA, MX, NS, TXT, etc). Hosted Zones are that the DNS system references. Hosted Zone is authoritative for a domain.

R53 Public Hosted Zone is a DNS database (zone file) hosted by R53 (Public Name Servers). Accessible from the public internet and VPCs. Hosted on 4 R53 name servers (NS) specific for the zone. Use NS records to point at these NS (connect to global DNS). Inside the hosted zone you create Resource Records (RR). Externally registered domains can point at R53 Public Zone.

R53 Private Hosted Zone is similar to R53 Public Hosted Zone, but is associated with VPCs and only accessible from these VPCs. Split-view (overlapping public and private) for public and internal use with the same zone name. A common architecture is to make the private zone a superset, containing more sensitive records.

## CNAME vs ALIAS

`A` record maps a name to an IP address (`catagram.io` → `1.3.3.7`). `CNAME` record maps a name to another name (`www.catagram.io` → `catagram.io`). `CNAME` is invalid for naked/apex (`catagram.io`). Many AWS services use a DNS name (for example, ELB). With just `CNAME` mapping `catagram.io` to ELB would be invalid.

`ALIAS` records map a name to an AWS resource and can be used for both naked/apex and normal records. There is no charge for `ALIAS` requests pointing at AWS resource. For AWS services pick `ALIAS` as default. Should be the same type as what the record is pointing at. For example, ELB has an `A`-record (maps name to IP address), so you need to use `A`-type `ALIAS`. Other examples of such services are:

- API Gateway
- CloudFront
- Elastic Beanstalk
- ELB (mentioned above)
- Global Accelerator
- S3

## Simple Routing

Supports 1 record per name, but each record can have multiple values (for example, `www` → `1.2.3.4`, `1.2.3.5`, `1.2.3.6`). Upon DNS request, all values are returned in **random** order; client chooses and uses 1 value. Simple Routing doesn't support health checks.

## Health Checks

Health checks are separate from, but are used by records. Health checkers located globally. Health checkers check every 30s (every 10s costs extra). Checks could be TCP, HTTP/HTTPS, HTTP/HTTPS with string matching over the response body. Health check status can be `Healthy` or `Unhealthy`. Checks can monitor:
- Endpoint
- State of CloudWatch alarm
- Status of other health checks (calculated health check)

Health check considered healthy if **18%+** of (globally distributed) health checkers report as healthy.

## Failover Routing

A common architecture is to use failover for "out of bound" failure / maintenance page for a service. You can add a primary and secondary records for the same name. Health check occurs on a primary record. If the target of the health check is healthy, the primary record is used. If the target of the health check is unhealthy, any queries return the secondary record of the same name.

## Multi Value Routing

Multiple records with the same name, each have an associated health check. Upon query, up to 8 (randomly selected) healthy records returned. Client chooses one to connect to the resource. Multi Value Routing improves availability, but it is not a replacement for load balancing.

## Weighted Routing

Simple load balancing or testing new software versions. Multiple records with the same name, each have a weight. Each record will be returned with frequency proportional to the ratio of this record's weight to a total weight of all the records with the same name. A record with 0 weight will never be returned (unless all other records also have 0 weight). If a chosen record is unhealthy, the process of selection is repeated until a healthy record is chosen.

## Latency-Based Routing

Used when optimizing for performance and user experience. You can specify a region for each record. Latency-based routing supports one record with the same name in each AWS region. AWS maintains a database of latency between the users general location and the regions tagged in records. The record returned is the one which offers the lowest estimated latency and is healthy.

## Geolocation Routing

Records are tagged with location (`<US state>`, `<country>`, `<continent>`, `<default>`). An IP check verifies the location of the user. Geolocation doesn't return "closest" records; it returns relevant (location) records. R53 checks for records:
1. In the `state`
2. In the `country`
3. On the `continent`
4. (optionally) `default`
and returns the most specific record or **no answer**.

Can be used for regional restrictions, language specific content or load balancing across regional endpoints.

## Geoproximity Routing

Records can be tagged with an AWS region or latitude & longitude coordinates. Record will be selected based on the physical distance between the user and the record. Also, a bias can be assigned to a record: `+` increases region size and decreases neighboring regions. This bias will be considered in distance calculation.

## Interoperability

R53 normally has 2 jobs: domain registrar and domain hosting. It can do both or either. When registering a domain, R53:
- accepts your money (domain registration fee)
- allocates 4 name servers (NS)
- creates a zone file on the above NS
- communicates with the registry of the TLD
- sets the NS records for the domain to point at the 4 NS above

## Quiz
- What routing policy can be used to distribute load over recordsets in a controlled way
  - Weighted
- Which routing policy type can be used to implement simple high-availability
  - Failover *(note: strange, should be "Multi Value", IMO)*
- Which type of recordset is generally used to point at AWS resources
  - A + Alias
- What type of hosted zone is available only within 1 or more VPCs
  - Private Hosted Zones
- What feature of Route53 can help improve the availability of service to customers
  - HealthChecks