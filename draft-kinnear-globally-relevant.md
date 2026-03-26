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

normative:
  RFC9460:

informative:
  RFC9461:
  RFC4033:

--- abstract

DNS answers for SVCB and HTTPS resource records are typically treated as
scoped to the network on which they were obtained. This requires clients to
re-resolve DNS when changing network attachments, adding latency to connection
establishment. This document defines a new SvcParamKey, "globally-relevant",
for use in SVCB and HTTPS DNS resource records as defined in {{RFC9460}}. When
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

The following term is used in this document:

Network attachment change:
: A change in the client's network connectivity, such as switching from one
Wi-Fi network to another, transitioning between Wi-Fi and cellular
connectivity, or connecting or disconnecting a VPN.


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


# Security Considerations

## Incorrect Flagging

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
