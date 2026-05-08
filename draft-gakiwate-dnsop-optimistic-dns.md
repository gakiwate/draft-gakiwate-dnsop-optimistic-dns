---
title: "Optimistic DNS"
abbrev: "Optimistic DNS"
category: std
docname: draft-gakiwate-dnsop-optimistic-dns-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Domain Name System Operations"
keyword: DNS, caching, latency, stub resolver
venue:
  group: "Domain Name System Operations"
  type: "Working Group"
  mail: "dnsop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dnsop/"
  github: "gakiwate/draft-gakiwate-dnsop-optimistic-dns"
  latest: "https://gakiwate.github.io/draft-gakiwate-dnsop-optimistic-dns/draft-gakiwate-dnsop-optimistic-dns.html"

author:
 -
    fullname: Your Name Here
    organization: Apple Inc
    email: your.email@example.com

--- abstract

DNS lookups introduce user-visible latency, particularly when cached records
have expired and must be refreshed from the network.  This document describes
Optimistic DNS, a client-side stub resolver mechanism that immediately returns
expired cached DNS records to applications while simultaneously refreshing them
with a network query.  The application receives an answer in microseconds rather
than milliseconds, and if the data has changed receives an updated answer
shortly thereafter.  Optimistic DNS is complementary to RFC 8767, which
addresses serving stale data at recursive resolvers.  This document focuses
exclusively on client-side stub resolver behavior, including explicit signaling
between the stub resolver and applications about answer freshness.

--- middle

# Introduction

When a user opens their laptop and navigates to a website, the first step
is typically a DNS lookup to translate the hostname into an IP address.  If
the laptop's DNS cache contains a record for that hostname, the answer is
returned almost instantly and the user perceives no delay.  But if the
record's TTL has expired, even by a single second, the laptop must perform
a fresh DNS lookup.  On a wired connection this might take 50 to 200
milliseconds.  On a cellular connection, particularly at the edge of
coverage, it can take several seconds.

The user experiences this as a brief but noticeable pause.  The website
appears to hang.  The user wonders if something is wrong.  And in most
cases, the expired record still contained the correct IP address and the
website has not moved to a different server.  The user waited for
confirmation of something the device already knew.

Consider a record for www.example.com with a TTL of 60 seconds.  At time
T=0 the record is fetched and cached.  For the next 60 seconds, any
application that asks for www.example.com gets an instant answer.  At time T=61,
the record has expired.  The very next lookup must go to the network.  From the
user's perspective, the transition from "instant" to "slow" is a cliff edge. The
record was valid for 60 seconds, then invalid for the fraction of a second it
took to refresh it, then valid again.  Yet that fraction of a second is the one
the user noticed.

This document describes Optimistic DNS, a stub resolver mechanism that addresses
this problem.  When the stub resolver has expired cached records that match a
query, it returns those expired records to the application immediately while
simultaneously issuing a fresh network query in the background.  The application
first receives the expired (but likely still correct) records, and then the
authoritative fresh records if different.  Of course, once the application has
the expired address it will start connecting. Thus, the query needs to remain
open to asynchronously deliver the fresh answer when it arrives, and the
connection logic needs to cope if the expired address turns out to be wrong
({{enabling-technologies}}).

The following diagram illustrates the timing difference between
conventional DNS resolution and Optimistic DNS:

**Conventional DNS (after cache expiry):**

~~~
  Application        Stub Resolver        DNS Server
      |                    |                    |
      |--- Query --------->|                    |
      |                    |--- Query --------->|
      |                    |                    |
      |          (waiting for network)          |
      |                    |                    |
      |                    |<-- Response -------|
      |<-- Fresh Answer ---|                    |
      |                    |                    |
      [====== 150ms+ latency ======]
~~~

**Optimistic DNS (after cache expiry):**

~~~
  Application        Stub Resolver        DNS Server
      |                    |                    |
      |--- Query --------->|                    |
      |<- Expired Answer --|--- Query --------->|
      |   (~0ms latency)   |                    |
      |                    |                    |
      |  (app can start    |                    |
      |  connecting now)   |                    |
      |                    |<-- Response -------|
      |<-- Fresh Answer ---|                    |
      |    (if changed)    |                    |
~~~

Optimistic DNS is complementary to Serving Stale Data to Improve DNS
Resiliency {{!RFC8767}}, which allows recursive resolvers to serve stale data
during upstream failures.  The two mechanisms address different parts of the
resolution chain: RFC 8767 operates at the recursive resolver, while
Optimistic DNS operates at the stub resolver on the end-user's device.
Both can be deployed simultaneously for layered staleness tolerance.

This document describes only the client-side behavior.  Optimistic DNS does
not define any new DNS wire-protocol messages, opcodes, or EDNS options.
The signaling between the stub resolver and the application is purely a
local API matter.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

*Stub Resolver.*
: A DNS resolver that operates on the end-user device.  It maintains a
  local cache of DNS records and forwards queries to configured recursive
  resolvers when cached answers are unavailable or expired.  This is the
  component that implements Optimistic DNS.

*Expired Record.*
: A cached DNS record whose TTL has reached zero.  Under conventional DNS
  caching rules, such a record would be purged from the cache or ignored
  when answering queries.

*Optimistic Answer.*
: An expired cached record that is returned to an application in response
  to a query, before the stub resolver has confirmed whether the record's
  data is still current.

*Fresh Answer.*
: A DNS record received from the network in response to a query, as
  opposed to a record served from the local cache.  Fresh answers reflect
  the current state of the authoritative DNS data (subject to normal
  caching along the resolution path).

*TTL Stretching.*
: A stub resolver technique that transparently extends the effective
  lifetime of cached records beyond their original TTL.  The resolver continues
  to serve expired records to applications. Unlike Optimistic DNS, no background
  query is initiated and the application cannot opt-out.

*Asynchronous DNS Resolution.*
: A DNS resolution model where the application initiates a query and
  receives results through callbacks or event notifications as they
  become available, rather than blocking until a single answer is
  returned.  This model supports receiving multiple answers over time,
  including updated answers that supersede earlier ones.

*Happy Eyeballs.*
: A client-side connection establishment algorithm (defined in {{!RFC6555}},
{{!RFC8305}} and {{!I-D.ietf-happy-happyeyeballs-v3}}) that races connection
attempts across multiple addresses and address families, using whichever
connection succeeds first.  Failed connection attempts to individual addresses
are absorbed within the algorithm's normal timeout budget.

# Problem Statement

The DNS TTL mechanism creates an inherent tension between freshness and
performance.  When a record is cached and its TTL has not expired, lookups
are essentially free and the answer is returned from local memory in
microseconds.  The moment the TTL expires, the cost jumps to a full
network round trip. This is not a graceful degradation.  It is a cliff.

This problem is compounded by several factors:

*Short TTLs.*
: Many content delivery networks and cloud services use TTLs of 60 seconds
  or less to facilitate rapid failover and load balancing.  Short TTLs mean
  more frequent cache expiration events, which means users hit the latency
  cliff more often.

*High-latency networks.*
: On cellular networks, satellite links, or congested Wi-Fi, a DNS round
  trip can take one to five seconds.  The penalty for a cache miss is
  severe.

*Multiple queries per page load.*
: A typical web page load involves DNS queries for dozens of hostnames (the
  page itself, stylesheets, scripts, images, analytics, ads).  If several
  of these expire simultaneously, the cumulative delay is substantial.

*First query after sleep.*
: When a laptop or phone wakes from sleep, many cached records will have
  expired during the sleep period.  The first network operation after
  waking pays the maximum penalty, precisely when the user is most
  impatient.

The fundamental observation behind Optimistic DNS is that in most cases, a
DNS record that expired a little while ago still contains the correct data.
Servers do not typically change IP addresses the instant a TTL expires.
The TTL is a freshness hint, not a correctness deadline.

When used in conjunction with Asynchronous DNS Resolution and Happy Eyeballs,
there is little to no cost to using a stale answer that turns out to be wrong.
An application that receives an expired record and begins connecting to the
address it contains will, in the vast majority of cases, succeed.  Even in the
rare case where the address has changed, it is not fatal to the application
since the application can then try again with the new candidates when the
background query returns with the fresh answers.  Any security risk from
connecting to a wrong address can be mitigated by adopting transport-layer
security ({{security-considerations}}).

The cost of waiting for a fresh answer that confirms what the cache already had
is a guaranteed delay of the full network round-trip time on every cache expiry.
For most applications, the expected value of the optimistic approach is clearly
positive.

# TTL Stretching

TTL Stretching is a simple modification to the stub resolver behavior where it
unilaterally extends the effective lifetime of cached records.  When a record's
TTL expires, instead of immediately discarding it or refusing to serve it, the
resolver continues to return it to applications for a brief grace period --
seconds to minutes.  The application receives the record as if it were still
fresh.  No background query is initiated.  No new API is involved.  From the
application's perspective, the record simply has a longer TTL than the
authoritative server originally specified.

TTL stretching is attractive because it is entirely transparent.  Every
application benefits automatically, with no code changes.  The latency cliff
disappears: instead of a sudden jump from zero to hundreds of milliseconds, the
application continues to receive instant cache answers while the record remains
within its stretched lifetime.

However, TTL stretching has limitations that motivate the more
sophisticated Optimistic DNS mechanism:

*No application opt-out.*
: All applications receive stretched TTLs whether they want them or not.  An
application that specifically needs authoritative-fresh data (for example, to
verify that a DNS-based access control record has been updated) has no way to
bypass the stretched cache. As such, in absence of this visibility applications
are unable to make risk-appropriate decisions.

*Bounded effectiveness.*
: TTL stretching is practical only for brief periods of staleness.  A stretch
window of a few minutes handles the common case of a record expiring between
successive page loads.  Extending the stretch window significantly increases the
risk of returning incorrect data with no way for the application to detect it.

Optimistic DNS generalizes TTL stretching. The application receives expired
records just as quickly, but since the applications can opt-out, Optimistic DNS
to extend the mechanism to much longer time horizons while keeping the
application in control.

TTL stretching can be viewed as a simpler version of Optimistic DNS: one where
the application has no visibility, no background query is initiated, and the
stretch window must remain short because there is no mechanism for the
application to handle stale data
intelligently.

# Enabling Technologies {#enabling-technologies}

When an application receives an expired address and immediately begins
connecting to it, two things need to happen.  First, when the fresh
answer arrives moments later, the application needs a way to receive it
-- which means the DNS API cannot have already returned a single answer
and closed the query.  Second, if the expired address turns out to be
wrong, the application needs a way to recover.

## Asynchronous DNS Resolution

Traditional synchronous DNS APIs typically block until a single set of answers
is returned.  The caller issues a query, waits, receives one set of results, and
the call is complete.  Optimistic DNS cannot function with this model since
there is no mechanism to deliver an expired answer now and a fresh answer later.

With Asynchronous DNS APIs, the application registers a callback and receives
results as they become available.  The query remains active, and the resolver
delivers additional results through subsequent callbacks.  This is the
resolution model defined by Multicast DNS {{!RFC6762}}.

This model naturally supports the two-wave delivery that Optimistic DNS
requires.  Expired records arrive in the first callback, within
microseconds.  Fresh records from the network arrive in subsequent
callbacks, within milliseconds.  The application can act on the expired
answer immediately by opening a connection.  If the address has
changed, the application can start a new connection to the updated
address.  If the address is the same, the connection is already
established and the fresh answer serves as confirmation.

## Happy Eyeballs

Happy Eyeballs (Version 1 {{!RFC6555}}, Version 2 {{!RFC8305}} and their successor
Version 3 {{!I-D.ietf-happy-happyeyeballs-v3}}) define algorithms for racing
connection attempts across multiple addresses and address families.  When a
client has several candidate addresses for a destination, Happy Eyeballs
staggers connection attempts with short delays and uses whichever connection
succeeds first.  Failed attempts to individual addresses are absorbed within the
algorithm's normal timeout budget.

This mechanism pairs naturally with Optimistic DNS.  When the resolver
returns expired addresses, Happy Eyeballs can begin racing connections to
those addresses immediately, rather than waiting for DNS resolution to
complete before starting any connection attempt.

When the expired addresses are still correct (the most common scenario) a
connection succeeds before the fresh DNS answer even arrives.  When an expired
address is wrong (the rare scenario) the failed connection attempt is simply one
candidate among several.  Happy Eyeballs is already designed to tolerate some
addresses failing.  When the fresh DNS answer arrives with the correct address,
it enters the ongoing connection race.  The cost of the wrong expired address is
a single failed attempt and its associated network costs.

Without Happy Eyeballs, a wrong expired address means a failed connection and a
visible delay while the application falls back to the fresh answer and retries.

## Combined Effect

Asynchronous DNS resolution makes Optimistic DNS *possible*.  It provides the
delivery mechanism for two waves of results.  Happy Eyeballs makes Optimistic
DNS *safe*.  It ensures that acting on a wrong expired address is not fatal to
the overall connection attempt.  Together, the application starts connecting
instantly with best-effort cached addresses, the connection race handles any
staleness gracefully, and the fresh DNS answer arrives as an update confirming
or correcting the initial result.

# Optimistic DNS Overview

Building on the asynchronous resolution model and connection racing described
above, Optimistic DNS introduces a specific modification to stub resolver
behavior.  When an application that has opted in issues a DNS query and the
stub resolver's cache contains expired records matching that query, the resolver
performs two actions in parallel:

1. It immediately returns the expired cached records to the application.

2. It issues a standard DNS query on the network to obtain fresh records.

As fresh answers arrive from the network, the resolver delivers them to
the application through the asynchronous callback mechanism.  The
following diagram shows both waves for a query where www.example.com
has an expired A record (93.184.216.34) in the cache, and the fresh answer
returns the same address:

| Time   | Event                          | Notes                    |
|--------|--------------------------------|--------------------------|
| T+0us  | App queries www.example.com    |                          |
| T+5us  | Callback: 93.184.216.34        | Expired                  |
| T+120ms| (data unchanged, no callback)  | Fresh Answer Same        |

When the fresh answer matches the expired answer, no second callback is
delivered. The application that began connecting at T+5us saved 120 milliseconds
compared to waiting for the fresh answer.

When the data has changed (for example, if the server moved to a new
address), the sequence looks like this:

| Time   | Event                          | Notes                    |
|--------|--------------------------------|--------------------------|
| T+0us  | App queries www.example.com    |                          |
| T+5us  | Callback: 93.184.216.34        | Expired                  |
| T+120ms| Callback: 198.51.100.42        | Fresh Answer Different   |

Here the expired answer contained the old address.  An application that
connected to 93.184.216.34 may find that the connection fails or returns
unexpected content.  But the fresh answer arrives 120 milliseconds later,
and the application can retry with the correct address.  The total time to
a successful connection is still only about 120 milliseconds -- the same as
it would have been without Optimistic DNS.

Optimistic DNS never makes things worse for the application.  In the common
case where the data has not changed, it makes things dramatically faster.  In
the uncommon case where the data has changed, it costs nothing beyond a failed
connection attempt that overlaps with the network query the resolver would have
performed anyway.

Optimistic DNS is an opt-in mechanism.  Applications that do not request it
receive conventional stub resolver behavior: expired records are ignored,
and only fresh network answers are returned.  This ensures backward
compatibility and allows applications to choose the tradeoff that suits
their needs.

# Implementation Details

## Query Initiation

To use Optimistic DNS, an application signals its willingness to receive
expired answers when it issues a DNS query.  This signaling is a local
matter between the application and the stub resolver.

One possible mechanism is for the stub resolver API to provide a flag that the
application includes with its query.  Setting this flag indicates that the
application is prepared to handle expired answers and will treat them
appropriately.

## Cache Lookup with Expired Records

When the stub resolver processes a query, it iterates through its cache looking
for records that match the query's name, type, and class.  For each matching
record, it determines whether the record has expired by comparing the time
elapsed since the record was received against the record's original TTL.

If a matching record has expired:

- If the record is a positive record (contains actual resource record
  data), it is returned to the application with the expired flag set.

- If the record is a negative cache entry (representing a previous
  NXDOMAIN or NODATA response), it is NOT returned.  Negative cache
  entries are excluded from Optimistic DNS because returning a stale
  "this name does not exist" answer could prevent the application from
  discovering that the name has since been created.  The cost of a false
  negative (telling the application a name does not exist when it now
  does) is higher than the cost of a brief delay.

If a matching record has not expired, it is returned normally, as with any
cache hit.


## Parallel Network Query

If unexpired records are found in the cache, they are returned to the
application as a normal cache hit and no network query is needed.

If no unexpired records are found, the stub resolver issues a standard DNS
query on the network.  This query proceeds through the normal resolution path:
contacting configured recursive resolvers, following CNAME chains, appending
search domains if applicable, and so on.  This network query is not optional.
Even if expired records were already returned from the cache, the network query
MUST still be issued.  The expired records are a convenience for the
application, not a substitute for proper DNS resolution.

Fresh answers from the network are delivered to the application through the
normal callback mechanism.  The application can use these fresh answers to
confirm or replace any expired answers it received earlier.

## CNAME Handling {#cname-handling}

CNAME records introduce a complication for Optimistic DNS.  When a DNS
query encounters a CNAME record, the resolver must follow the CNAME chain
to find the ultimate answer.  If the CNAME record itself is expired,
following it may lead to a stale alias that no longer points to the correct
canonical name.

Consider the following example.  An application queries for
www.example.com, which has a CNAME record pointing to cdn.example.net.
Both records are in the cache but expired:

~~~
  www.example.com.   CNAME  cdn.example.net.  (expired)
  cdn.example.net.   A      198.51.100.42     (expired)
~~~

If the resolver follows the expired CNAME to cdn.example.net and returns
the expired A record, the application receives a result.  But if the CNAME
has changed (www.example.com now points to cdn2.example.net instead), the
resolver has followed a stale chain and returned a record for the wrong
name.

To handle this safely, when the stub resolver encounters an expired CNAME
record during optimistic resolution, it takes the following steps:

TODO: Phil to fill in the steps

This rewind-and-restart approach ensures that the application always
receives a complete, consistent answer from the fresh network query, even
if the CNAME chain has changed since the cached records were stored.

## Search Domains

When the stub resolver appends search domains to partially-qualified domain
names, Optimistic DNS interacts with the search domain iteration process.

If an optimistic query encounters an expired CNAME while using a particular
search domain, the query restart described in {{cname-handling}} preserves
the current search domain.  The restart does not advance to the next search
domain in the list, because the current search domain may still be the
correct one -- the expired CNAME merely prevented the resolver from
following the chain to a fresh answer.

For example, if the user queries for "mail" and the search domain list is
\[corp.example.com, example.com\]:

1. The resolver tries mail.corp.example.com with Optimistic DNS.
2. An expired CNAME is found for mail.corp.example.com pointing to
   mailserver.corp.example.com.
3. The resolver restarts at mail.corp.example.com (not mail.example.com).
4. The fresh network query resolves mail.corp.example.com normally.

# Interaction with Other DNS Features

## DNSSEC {#dnssec}

Optimistic DNS MUST NOT be used for DNSSEC-validated responses.

DNSSEC relies on cryptographic signatures (RRSIG records) that have
explicit validity periods.  An expired DNS record may have an associated
RRSIG whose signature validity period has also expired.  Serving such a
record as an optimistic answer would provide the application with data
whose cryptographic authentication can no longer be verified, undermining
the security guarantees that DNSSEC is designed to provide.

## Encrypted DNS Transports

Optimistic DNS is transport-agnostic.  It operates entirely within the stub
resolver's cache layer, which sits above the transport layer.  Whether the
stub resolver communicates with recursive resolvers using classic DNS over
UDP/TCP, DNS over TLS (DoT) {{!RFC7858}}, DNS over HTTPS (DoH)
{{!RFC8484}}, or DNS over QUIC (DoQ) {{!RFC9250}}, the Optimistic DNS
mechanism functions identically.

## Relationship to RFC 8767

Serving Stale Data to Improve DNS Resiliency {{!RFC8767}} describes a
mechanism for recursive resolvers to serve stale cached data when they are
unable to refresh it from authoritative servers.  Optimistic DNS and
RFC 8767 address different levels of the DNS resolution chain:

RFC 8767
: Operates at the recursive resolver.  Serves stale data when upstream
  authoritative servers are unreachable.  The primary goal is resiliency --
  maintaining DNS service during outages.

Optimistic DNS
: Operates at the stub resolver on the end-user's device.  Serves expired
  cached data proactively, before attempting a network query.  The primary
  goal is latency reduction -- eliminating the TTL expiry cliff.

The two mechanisms are complementary and can be deployed simultaneously.
When both are active, the resolution chain has two layers of staleness
tolerance:

1. The stub resolver returns expired records optimistically, eliminating
   client-perceived latency.

2. If the recursive resolver's cache has also expired and the authoritative
   server is unreachable, the recursive resolver can serve its own stale
   data rather than returning SERVFAIL.

RFC 8767 serves stale data transparently, making it more similar to TTL
stretching ({{ttl-stretching}}) than to Optimistic DNS. By contrast, Optimistic
DNS explicitly empowers the application to make informed decisions.

# Security Considerations {#security-considerations}

Optimistic DNS introduces a tradeoff between latency and freshness.
Applications that use expired answers must be prepared for those answers to be
incorrect.  Applications using Optimistic DNS SHOULD employ transport-layer
security (TLS, QUIC) when connecting to addresses obtained from expired answers,
to detect cases where the address has been reassigned.

IP Address Reuse
: When a DNS record expires, the IP address it contained may no longer be
  associated with the original domain.  If the address has been reassigned
  to a different entity, an application connecting to that address may
  reach the wrong server.  For HTTPS connections, TLS certificate
  validation will detect this mismatch and prevent data from being sent to
  the wrong party.  For unencrypted protocols, there is a risk of
  connecting to an unintended server.

Poisoning Amplification
: If an attacker successfully poisons a cache entry, Optimistic DNS could
  extend the lifetime of the poisoned record beyond its original TTL.
  However, the record will be purged after the retention period expires.
  Moreover, the parallel network query will obtain and deliver the correct
  answer, giving the application an opportunity to detect the discrepancy.

DNSSEC Exclusion
: Optimistic DNS excludes DNSSEC-validated records from extended caching
  and expired serving.  This prevents the serving of records whose
  cryptographic signatures have expired, maintaining DNSSEC's security
  guarantees.

Expired Records Retention Period
: The retention period (recommended maximum of one week) bounds the
  maximum time a stale record can be served.  Implementations SHOULD allow
  this period to be configured.  Shorter periods reduce the window of
  exposure to stale data but also reduce the effectiveness of Optimistic
  DNS for infrequently-accessed names.


# IANA Considerations

This document has no IANA actions.


--- back

# Deployment History

Optimistic DNS was first implemented in mDNSResponder in January 2018 and
shipped enabled by default in macOS 10.14 (Mojave) and iOS 12 in September 2018.
It has been active on all Apple platforms since that release, serving as the
default stub resolver behavior for all applications that use asynchronous DNS
resolution APIs.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
