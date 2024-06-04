# DNS & DNSSEC

DNS main role is to resolve host names into IP addresses.

DNS needs complex hierarchical structure to:
- protect itself from potential attackers
- scale accordingly to the number of clients
- support a huge amount of data (DNS records): ~341 million domains at the moment

## Key terms
- DNS `Zone`: a database, containing records
  - example: `*.netflix.com`
  - stored in the `ZoneFile`
- `Name Server` (`NS`): a DNS server which hosts 1 or more `Zones` and stores 1 or more `ZoneFiles`; `NS` could be:
  - `Authoritative`: contains real / genuine records
  - `Non-Authoritative` (`Cached`): contains copies of records/zones stored to speed things up

## DNS architecture
- DNS Root
  - 13 geographically distributed IP addresses (NASA, universities, etc.)
  - IANA is responsible for managing this zone
  - not much data: high-level information on top-level domains (TLDs) like `.com` and `.ua`
  - delegates TLD management to other zones/companies (for example, `.com` is managed by Verisign)

## How DNS works

The job of DNS is to help you locate and get a query response from the authoritative zone which hosts the DNS record(s) you need.

Order of DNS records lookup ("walking the tree"), example for `www.netflix.com`:
- checking local computer cache
- checking local computer hosts file
- local computer sends query to DNS Resolver (running on router or within the internet provider)
- DNS Resolver checks local cache (non-authoritative)
- DNS Resolver sends query to DNS Root
- (DNS Root trusts `.com` `NS`)
- DNS Root returns trusted `NS` for a specific TLD (in this example, `.com`)
- DNS Resolver asks `.com` `NS` for `www.netflix.com`
- (TLD `NS` trusts `NS` of a specific domain, in this example `netflix.com`)
- TLD `NS` returns `netflix.com` `NS` to DNS Resolver
- DNS Resolver queries `netflix.com` `NS` for `www.netflix.com`
- `netflix.com` `NS` returns authoritative DNS record for `www.netflix.com` to DNS Resolver
- DNS Resolver caches the IP for `www.netflix.com`
- DNS Resolver returns the result to the local computer

In the example above, three main steps are:
- fetch nameservers for `.com` from Root Zone
- fetch nameservers for `netflix.com` from `.com` Zone
- fetch IP for `www.netflix.com` from `netflix.com` Zone

## Registering a domain

Main entities:
- person, registering the domain
- Domain Registrar (R53 registered domains): allows you to purchase domains
- DNS Hosting Provider (R53 hosted zones): operates DNS nameservers
- TLD Registry
- TLD Zone (managed by TLD Registry)

Steps required for registering a domain:
- check that domain is available and pay for it to registrar
- create a zone, get NS
- registrar informs TLD Registry
- TLD Registry adds domain's NS to TLD Zone

## DNSSEC

### Basics

Secure addon for DNS. DNS chain of trust between DNS zones. Benefits:
- data origin authentication (to be sure that the data is really from the specific zone)
- data integrity protection (to be sure that the data hasn't been modified in transit)

Basic DNS-compatible device will receive DNS results only. DNSSEC-compatible device will receive DNS + DNSSEC results. DNSSEC results will be used to validate (origin and integrity) the DNS results. This makes much harder, for example, to "poison" the DNS Resolver cache by fake responses.

Example of DNS query via the terminal:
```sh
dig www.icann.org
```

Example of DNS+DNSSEC query via the terminal:
```sh
dig www.icann.org +dnssec
```

### Zone

`RRSET` (resource record set): any DNS records of the same name and the same type. For example, an `RRSET` of 4 `MX` type records for `icann.org` name inside the zone `icann.org`.

The private part of `ZSK` (zone signing key) used to encrypt the `RRSET` to recieve the `RRSIG` records. Inside the DNSSEC Zone you will find the `RRSET`, `RRSIG` and `DNSKEY` (public part of `ZSK`) records. `DNSKEY` record can also store the public part of `KSK` (key signing key). `ZSK` and `KSK` differs by a special flag value (`256` for `ZSK` and `257` for `KSK`).

DNSSEC Resolver can take `RRSET`, `RRSIG` and `DNSKEY` and validate the `RRSET`'s origin and integrity.

`DNSKEY` record also has an `RRSIG` record, which is signed by `KSK` (key signing key). Parent zone links to the public part of the `KSK`. So, `ZSK` could be changed without updating parent zone(s).

### Chain of Trust

Example: `.org` zone needs to trust `icann.org` zone. To do that, `.org` zone stores the hash of the public `KSK` from `icann.org` in `DS` (delegate sign) type records. There is also `RRSIG DS` records in `.org` zone, which stores digital signature of `RRSET` of `DS` records (signed by `.org` `ZSK`). The same goes for Root zone -> `.org` zone trust.

### Root Signing Ceremony

There is no parent zone for the Root zone, so no zone to ensure the trust of the Root zone's `KSK` (and, consequentially, the whole DNS chain). The trust anchor: Private DNS Root `KSK` ("key to the internet"). Stored in two different locations, locked away, protected & never exposed (Hardware Security Module). The public DNS Root `KSK` verifies the absolute trust (so, it is known by everybody).

The ceremony is signing ZSK with KSK and producing the RRSIG DNSKEY records in the Root zone. This happens approximately every 3 months. See [this article](https://www.cloudflare.com/dns/dnssec/root-signing-ceremony/) for details.
