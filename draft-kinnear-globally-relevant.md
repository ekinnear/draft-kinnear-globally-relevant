---
title: "Globally Relevant HTTPS RRs"
abbrev: "Globally Relevant HTTPS RRs"
category: std

docname: draft-kinnear-globally-relevant-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
keyword:
 - DNS
 - SVCB
 - HTTPS
 - SvcParamKey
 - globally-relevant
venue:
  github: "ekinnear/draft-kinnear-globally-relevant"
  latest: "https://ekinnear.github.io/draft-kinnear-globally-relevant/draft-kinnear-globally-relevant.html"

author:
 -
    fullname: "Eric Kinnear"
    organization: Apple Inc.
    email: "ekinnear@apple.com"

--- abstract

DNS answers for SVCB and HTTPS resource records are typically treated as
scoped to the network on which they were obtained. This requires clients to
re-resolve DNS when changing network attachments, adding latency to connection
establishment. This document defines a new SvcParamKey, "globally-relevant",
for use in SVCB and HTTPS DNS resource records as defined in {{!RFC9460}}. When
present, this boolean flag indicates that the service binding parameters in
the record are valid regardless of the client's network attachment point.
Clients that observe this flag can reuse cached SVCB and HTTPS records across
network changes, subject to normal TTL expiry.


--- middle

# Introduction

When a client changes its network attachment, for example, by switching from one
Wi-Fi network to another, transitioning between Wi-Fi and cellular
connectivity, or connecting or disconnecting a VPN, current practice requires
it to re-resolve DNS records for connections on the new network attachment.
This is because DNS answers may be network-specific due to split-horizon DNS
deployments, geographic load balancing, or network-level content policies. The
re-resolution adds latency to connection establishment after every network
transition, delaying the user experience.

Many widely-used services, however, serve DNS answers that are identical
regardless of which network the query traverses. Their HTTPS and SVCB resource
records, including ALPN protocol identifiers, Encrypted ClientHello
(ECH) configurations, port numbers, and address hints, are globally consistent.
For these services, re-resolving DNS after a network change is unnecessary
overhead that delays connection establishment without providing any benefit.

This document defines a new SvcParamKey called "globally-relevant" for SVCB
and HTTPS resource records {{RFC9460}}. It is a boolean flag with an empty
value that authoritative DNS servers set to indicate the record's service
binding parameters are valid regardless of network context. Clients observing
this flag MAY continue using cached SVCB and HTTPS records after network
transitions, avoiding the latency of re-resolution.

This mechanism is strictly opt-in. Services that do not include the
"globally-relevant" SvcParamKey continue with current behavior, and their
records are treated as network-scoped as they are today.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used in this document:

Network attachment change:
: A change in the client's network connectivity, such as switching from one
Wi-Fi network to another, transitioning between Wi-Fi and cellular
connectivity, or connecting or disconnecting a VPN.

Authoritative server:
: A DNS server that is authoritative for a zone and generates responses
for names in that zone, as defined in {{?RFC9499}}.

Recursive resolver:
: A DNS resolver that resolves queries on behalf of a client by
iteratively querying authoritative servers, as defined in {{?RFC9499}}.

Origin server:
: The server that terminates the client's transport connection to the
service identified by an SVCB or HTTPS resource record. For HTTP-based
services, this is the origin server as defined in
{{Section 3.6 of ?RFC9110}}. The `ipv4hint` and `ipv6hint` parameters of
an SVCB or HTTPS record are hints for reaching this server.

Operator:
: In this document, an entity that runs both the authoritative server for
a service's zone and the origin server for that service. Where a single
operator holds both roles, it can correlate observations between DNS
queries and connections.


# The "globally-relevant" SvcParamKey

The "globally-relevant" SvcParamKey is a boolean flag parameter for SVCB and
HTTPS resource records {{RFC9460}}. Its presence indicates that the
authoritative DNS server asserts the service binding parameters in this record
are valid regardless of the client's network attachment point.

## Wire Format

The SvcParamKey number is TBD. The value MUST be empty (length 0), following the
same pattern as "no-default-alpn" (SvcParamKey 2) defined in Section 7.1.2 of
{{RFC9460}}. If a client receives this SvcParamKey with a non-empty value, the
client MUST ignore the parameter.

## Presentation Format

In DNS zone file presentation format, the key is represented as simply
`globally-relevant` with no value. For example:

~~~ dns-rr
example.com. 300 IN HTTPS 1 . alpn=h2,h3 globally-relevant
~~~

## Applicability

This parameter is applicable to both SVCB (RR type 64) and HTTPS (RR type 65)
resource records. It is only meaningful in ServiceMode records (those with
SvcPriority greater than 0) that carry service binding parameters. It has no
meaning in AliasMode records.


# Client Behavior

When a client receives an SVCB or HTTPS record containing the
"globally-relevant" SvcParamKey, it MAY retain the cached record and continue
using its service parameters after a network attachment change.

The record's TTL still applies. The "globally-relevant" flag does not extend the
record's cache lifetime; it only permits reuse of the cached record across
network changes within the remaining TTL window.

If a client experiences connection failures when using cached parameters after a
network change, it SHOULD re-resolve the record regardless of the
"globally-relevant" flag.

Clients MAY choose to ignore the "globally-relevant" flag entirely and always
re-resolve on network changes. This flag is purely permissive and does not
mandate any specific client behavior.

When address hints (`ipv4hint`, `ipv6hint`) are present in a record carrying
the "globally-relevant" flag, the globally-relevant assertion covers those
hints as well. Clients MAY use the cached address hints on the new network,
though they SHOULD prefer fresh address resolution if readily available and MAY
choose to re-resolve in parallel with connection attempts to the cached address
hints.

## Address Synthesis Across Networks

On IPv6-only networks that provide NAT64 connectivity {{?RFC6146}}, clients can
synthesize IPv6 addresses from IPv4 addresses using the network's NAT64 prefix,
discovered via Router Advertisements {{?RFC8781}} or DNS-based mechanisms
{{?RFC7050}}. The "globally-relevant" flag asserts that the addresses in
`ipv4hint` and `ipv6hint` are globally valid as published by the authoritative
server; it does not assert the validity of any locally-synthesized addresses
derived from those hints.

If a client has synthesized an IPv6 address from an `ipv4hint` value using one
network's NAT64 prefix, it MUST NOT reuse that synthesized address after moving
to a different network. Instead, the client MUST re-synthesize using the NAT64
prefix of the new network, if available. The original `ipv4hint` value itself
remains valid and can be used as input to the new synthesis.

# Server Behavior

An authoritative DNS server SHOULD set the "globally-relevant" SvcParamKey
only when the service binding parameters in the record are consistent across
all resolvers and network paths. This typically applies to globally-deployed
services with uniform configurations.

Services that employ split-horizon DNS, geo-dependent load balancing, or
network-specific configurations MUST NOT set this flag. The flag asserts that
all parameters in the record, including ALPN values, ECH configurations, and
port numbers, are globally valid. If any parameter varies by network context,
the flag MUST NOT be set.

In particular, if the record includes `ipv4hint` or `ipv6hint` parameters,
the addresses contained in those hints MUST be reachable and correct on all
networks, not just the network on which the record was originally resolved.
Servers that use address hints to direct clients to network-specific
endpoints (e.g., CDN edge nodes selected by resolver location) MUST NOT set
the "globally-relevant" flag unless those address hints are valid globally.

Operators SHOULD carefully audit their DNS configurations before deploying
the "globally-relevant" flag, as incorrect use could cause clients to use
inappropriate service parameters after network changes.

# Resolver Behavior

Recursive resolvers MUST pass the "globally-relevant" SvcParamKey through
to clients transparently, without modification.

Resolvers MUST NOT add or remove the "globally-relevant" flag. The assertion
of global relevance is made by the authoritative server for the zone, and
resolvers are not in a position to make or override this determination.

Some resolvers modify DNS responses for operational purposes, such as DNS64
synthesis {{?RFC6147}} or content filtering. When a resolver synthesizes or
rewrites a response, it is effectively acting as the authoritative source for
that modified answer. Such synthesized answers SHOULD NOT carry
the "globally-relevant" flag, as the resolver cannot assert that its
locally-modified response is valid on other networks.


# Security Considerations

## Incorrect Flagging {#incorrect-flagging}

If an authoritative server incorrectly sets the "globally-relevant" flag on a
record whose parameters vary by network, clients may use inappropriate service
configurations after a network change. This could manifest as connection
failures or as connections to unintended endpoints if address hints are
incorrect for the client's new network. Operators bear responsibility for
ensuring the flag is only set on records with truly globally-consistent
parameters.

Clients MUST NOT reuse cached address hints across network changes for
connections that are not authenticated by a security protocol, such as TLS.
This allows clients to reject connections established to an address that
responds to incoming packets, but no longer represents the desired host.


## Network Filtering

Some networks apply DNS-based content filtering or access control policies.
The "globally-relevant" flag allows clients to skip DNS re-resolution when
joining such networks, which means the network's filtered DNS responses would
not be applied to cached records. However, this does not meaningfully weaken
network-level controls: users can already bypass DNS-level filtering by using
alternative resolvers, encrypted DNS, or connecting directly to known IP
addresses in some cases. Networks that require effective traffic control
already need to enforce policies at layers beyond DNS, such as IP-level or
SNI-based firewalling. The "globally-relevant" flag does not change this.


## DNSSEC Considerations

Where DNSSEC validation is employed, the "globally-relevant" flag does not
change validation requirements. Cached records that were validated remain
usable as long as the DNSSEC signatures have not expired. Clients performing
DNSSEC validation SHOULD NOT reuse a cached record if the DNSSEC signature has
expired, even if the record's TTL has not.


## Downgrade and Upgrade Attacks

An on-path attacker that can modify DNS responses could strip the
"globally-relevant" flag from records, causing clients to re-resolve
unnecessarily after network changes. This degrades performance to current
behavior but does not otherwise affect security posture.

An on-path attacker could also add the "globally-relevant" flag to a record that
the authoritative server did not mark as globally relevant. The impact of this
is limited: the client may reuse a record that would otherwise have been
re-resolved, which is only problematic if the record was network-specific, and
the resulting connection will fail to authenticate the host. DNSSEC also
protects against both of these modification attacks when deployed.


# Privacy Considerations

## Cross-Network Client Tracking via Unique Addresses

The "globally-relevant" flag permits a client to use cached `ipv6hint` (or
`ipv4hint`) addresses across network attachment changes. An authoritative
server that wishes to track a specific client across networks could exploit
this by returning a per-client unique address, for example by encoding a
client identifier in some of the bits of an IPv6 address, and observing
the resulting connections from multiple networks. Because the client reuses
the cached address after a network change, the server observes the same
unique address connecting from a different network and links the two
networks to a single client identity.

Without "globally-relevant", the client would re-resolve on the new
network, and the authoritative server would issue a fresh unique address
to that resolution. The connections originating from each network would
then carry different addresses, and the authoritative server would have no
DNS-derived signal linking them as the same client: each resolution arrives
through a different recursive resolver and conveys no carry-over client
identity. The cross-network linkage exists in this design precisely because
the unique address itself acts as a stable handle that the client carries
across networks instead of being replaced at each resolution.

{{Section 7.1 of ?RFC9076}} recognizes a related class of attack in which
a user is "re-identified via DNS queries... regardless of the location from
which the user makes those queries", based on query-pattern correlation
across time. The cross-network linkage described here arrives at the same
end through a different mechanism: rather than correlating recurring query
patterns observed by a passive watcher, an authoritative server seeds a
chosen identifier into the response itself and recognizes it whenever the
client connects to the resulting address, with no further DNS interaction
required.

## Comparison to Existing Tracking Vectors

### Recursive Resolvers and Anonymity Sets

An authoritative server that returns a per-client unique answer today
already observes that answer being requested by a specific recursive
resolver. For clients using a shared public or ISP recursive resolver, the
authoritative server sees only the recursive's address, and the client is
part of an anonymity set composed of that recursive's other users (see
{{Section 6.2 of ?RFC9076}}). The set's size depends on the deployment
(see {{Section 6.1.1 of ?RFC6973}} for definitions of anonymity sets).

This "hiding" by the recursive resolver is incomplete where the EDNS Client
Subnet (ECS) option {{?RFC7871}} is used. {{Section 6.2 of ?RFC9076}} notes
that with ECS, the authoritative name server "sees the original IP address
(or prefix, depending on the setup)" rather than only the recursive's
address. An authoritative server returning a per-client unique answer can
therefore correlate that answer with the client's subnet on each fresh
resolution, providing some cross-network correlation today even without
the "globally-relevant" flag. The flag broadens this in two directions: it
eliminates the dependence on ECS, and it removes the requirement for a
fresh resolution at all.

Without the "globally-relevant" flag, the authoritative server's view is
partitioned by recursive resolver: it sees one anonymity set per recursive,
and a single client moving between networks that use different recursives
appears as independent observations from disjoint sets. Caching at each
recursive further limits how often the authoritative server is queried at
all.

The "globally-relevant" flag changes this in two ways. First, when a client
moves to a new network and reuses a cached unique address, it bypasses the
new network's recursive resolver entirely for the cached name; the
authoritative server learns of the move directly from the connection,
without the new recursive ever forwarding a query. Second, this allows the
authoritative server to join observations that would otherwise have been
separated by recursive boundaries, effectively intersecting the
per-recursive anonymity sets and shrinking the set in which any one client
is hidden. {{Section 5.2.1 of ?RFC6973}} characterizes this kind of
correlation across observations as a privacy harm independent of any single
observation's identifying power.

### AliasMode and CNAME Targets

A similar tracking primitive is already available without this draft.
Authoritative servers can return per-client unique CNAME targets, SVCB
AliasMode targets {{Section 2.4.2 of RFC9460}}, or other names that
function as client identifiers. However, the "globally-relevant" attack
described here is harder to mount than its name-based analogues, for a
practical reason: a unique IPv4 or IPv6 address used as an `ipv4hint` or
`ipv6hint` value must actually be routable to the operator's service from
every network the client uses, whereas a unique CNAME or AliasMode target
is merely a label that the client subsequently resolves through its local
recursive resolver, ultimately pointing at a shared address that the
operator already serves.

The label-based approach therefore costs the operator only a DNS record per
identifier, while the address-based approach costs an IP address per
identifier and the routing infrastructure to receive traffic on each.
Operators with large unique-IPv6 ranges and the willingness to terminate
arbitrary addresses can perform this attack regardless. For the broader
population of operators, the deployment cost is a meaningful barrier.

The label-based primitive, however, is not bounded. An arbitrarily long
CNAME (or DNAME-induced CNAME) chain reintroduces the same tracking
capability, because the final A or AAAA record in the chain can encode a
per-client identifier in either its name or its address, and clients
caching across networks would carry the chain's terminal answer with them.
The DNS specifications {{?RFC1034}} {{?RFC6672}} do not normatively limit
CNAME chain length; {{Section 2.3 of ?RFC6672}} explicitly notes that
"fairly lengthy valid chains" may occur. Implementations bound chain
following for resource reasons, but the limit is not interoperable.
Treating CNAME chains as out-of-scope for this draft is appropriate, but
implementations of "globally-relevant" SHOULD apply the same cross-network
caching policy to all elements of an alias chain consistently, reusing
either all records in the chain after a network change or none, to avoid
creating partial-reuse states that have privacy properties differing from
either endpoint of the chain.

### Encrypted DNS and ODoH

Encrypted transports such as DNS-over-TLS {{?RFC7858}}, DNS-over-HTTPS
{{?RFC8484}}, and Oblivious DNS-over-HTTPS {{?RFC9230}} protect the DNS
query against on-path observers but do not prevent an authoritative server
from returning a per-client unique answer. {{Section 5.2 of ?RFC9076}}
notes that with encrypted transport "some privacy attacks are still
possible", and {{Section 6.1.4.1 of ?RFC9076}} observes that "use of
encrypted transports does not reduce the data available in the recursive
resolver". The same principle applies upstream: encrypted transport does
nothing to constrain what an authoritative server chooses to put in its
answers. ODoH in particular hides the
client's IP address from the recursive resolver (the "Target" in
{{Section 4 of ?RFC9230}}) and the query contents from the proxy, but its
threat model treats authoritative-server-as-tracker as out of scope: a
Target or any downstream authoritative server "could return a DNS answer
corresponding to an entity it controls and then observe the subsequent
connection from a Client" {{Section 11 of ?RFC9230}}. The
"globally-relevant" flag does not change this on a single network. It
extends the same attack surface across networks, subject to the constraints
described above.

## Client Mitigations

Clients implementing this specification SHOULD apply the following
mitigations to limit the cross-network tracking risk:

* Treat `ipv4hint` and `ipv6hint` values with low natural reuse as
  candidates for re-resolution after a network change, even when the
  "globally-relevant" flag is present. Examples include addresses outside
  well-known anycast ranges, addresses that appear unique to a single
  client, and addresses with high entropy in the host bits.

* When re-resolving a globally-relevant record on a new network in the
  background (for example, to refresh before TTL expiry), if the new
  network blocks or refuses the resolution, stop using the cached record
  rather than continuing to connect to the previously cached address. A
  failed refresh on a network that otherwise resolves DNS is a signal that
  the cached parameters may not be appropriate for the new network,
  regardless of the flag.

* Apply existing privacy protections for address hints uniformly to
  globally-relevant records, including the requirement in
  {{incorrect-flagging}} that cached address hints not be reused across
  network changes for connections that are not authenticated by a security
  protocol such as TLS.

The operational practice of long-TTL anycast addressing combined with
optimistic DNS refresh {{?RFC8767}} already produces
similar cross-network address reuse for many records today. The mitigations
above apply equally well in that context and are consistent with current
operational practice.


# IANA Considerations

This document requests IANA to register the following entry in the "Service
Parameter Keys (SvcParamKeys)" registry {{RFC9460}}:

| Number | Name              | Meaning                                   | Change Controller  | Reference      |
|--------|-------------------|-------------------------------------------|--------------------|----------------|
| TBD    | globally-relevant | Record is valid across network changes    | IETF               | (this document)|


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
