---
title: "HTTP Access Service Description Objects"
abbrev: "HTTP Access Descriptions"
category: std

docname: draft-schwartz-masque-access-descriptions-latest
v: 3
area: "Transport"
workgroup: "Multiplexed Application Substrate over QUIC Encryption"
keyword: Internet-Draft
venue:
  github: "bemasc/access-services"

author:
 -
    fullname: Benjamin M. Schwartz
    organization: Google LLC
    email: bemasc@google.com

normative:

informative:


--- abstract

HTTP proxies can operate several different kinds of access services.  This specification provides a format for identifying a collection of such services.


--- middle

# Introduction

In HTTP/1.1, forward proxy service was originally defined in two ways: absolute-uri request form (encrypted at most hop-by-hop), and HTTP CONNECT (potentially encrypted end-to-end).  Both of these services were effectively origin-scoped: the access service was a property of the origin, not associated with any particular path.

Recently, a variety of new standardized proxy-like services have emerged for HTTP.  These new services are defined by a URI template or path, allowing distinct instances of the same service type to be served by a single origin.  These services include:

* DNS over HTTPS {{!RFC8484}}
* CONNECT-UDP {{!I-D.draft-ietf-masque-connect-udp}}
* CONNECT-IP {{!I-D.draft-ietf-masque-connect-ip}}
* Oblivious HTTP {{!I-D.draft-ietf-ohai-ohttp}}

This specification provides a unified format for describing a collection of such access services, and a mechanism for reaching such services when the initial information contains only an HTTP origin.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Format

An access service collection is defined by a JSON dictionary containing keys specified in the corresponding registry ({{iana}}).  Inclusion of each key is OPTIONAL.  The corresponding media type is `application/access-services+json`.

The "dns", "udp", and "ip" keys are each defined to hold a JSON dictionary containing the key "template" with a value that is a URI template suitable for configuring DNS over HTTPS, CONNECT-UDP, or CONNECT-IP, respectively.

The "ohttp" key contains a dictionary with either or both of these keys:

* "relay", containing a dictionary with a "template" key indicating the Oblivious Relay's resource mapping.  The template MUST contain a "gateway_uri" variable indicating the Oblivious Gateway Resource.
* "gateway", containing a dictionary with a "uri" key indicating the Oblivious Gateway Resource and a "key" key conveying its KeyConfig in base64.

If the Access Description is for a general-purpose proxy, all Oblivious Gateways and proxy targets (respectively) are presumed to be supported; otherwise the supported Gateways and targets must be understood from context (but see {{well-known}}).

## Examples

~~~JSON
{
  "dns": {
    "template": "https://doh.example.com/dns-query{?dns}",
  },
  "udp": {
    "template":
        "https://proxy.example.org/masque{?target_host,target_port}"
  },
  "ip": {
    "template": "https://proxy.example.org/masque{?target,ip_proto}"
  },
  "ohttp": {
    "relay": {
      "template": "https://proxy.example.org/ohttp{?gateway_uri}"
    }
  }
}
~~~
{: title="A proxy with UDP, IP, DNS, and Oblivious HTTP support"}

~~~JSON
{
  "dns": {
    "template": "https://doh.example.com/dns-query{?dns}",
  },
  "ohttp": {
    "gateway": {
      "uri": "https://example.com/ohttp/",
      "key": "(KeyConfig in Base64)"
    }
  }
}
~~~
{: title="An Oblivious DNS over HTTPS service"}

# Discovery from an Origin {#well-known}

In cases where the HTTP access service is identified only by an origin (e.g. when configured as a Secure Web Proxy), operators can publish an associated access service collection at the path "/.well-known/access-services", with the Content-Type "application/access-services+json".

When the "ohttp.gateway" URI appears in an Access Description at this location, all URIs on this origin (except the Oblivious Gateway URI) are presumed to be reachable as Oblivious Targets.

Clients MAY fetch this Access Description and use the indicated services (in addition to any origin-scoped services) automatically.  Clients SHOULD use the description only while it is fresh according to its HTTP cache lifetime, refreshing it as needed.


# Security Considerations

TODO Security


# IANA Considerations {#iana}

IANA is requested to open a Specification Required registry entitled "HTTP Access Service Descriptors", with the following initial contents:

| Key   | Specification   |
|-------|-----------------|
| dns   | (This document) |
| udp   | (This document) |
| ip    | (This document) |
| ohttp | (This document) |

IANA is requested to add the following entry to the "Well-Known URIs" registry:

| URI Suffix      | Change Controller | Reference       | Status      | Related Information |
| --------------- | ----------------- | --------------- | ----------- | ------------------- |
| access-services | IETF              | (This document) | provisional | Sub-registry at (link)      |

IANA is requested to add the following entry to the "application" sub-registry of the "Media Types" registry:

| Name                 | Template                         | Reference       |
| -------------------- | -------------------------------- | --------------- |
| access-services+json | application/access-services+json | (This document) |

> TODO: Full registration template for this Media Type.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
