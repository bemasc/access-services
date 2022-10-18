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

Recently, a variety of new standardized proxy-like services have emerged for HTTP.  These new services are defined by a URI template, allowing distinct instances of the same service type to be served by a single origin.  These services include:

* DNS over HTTPS {{!RFC8484}}
* CONNECT-UDP {{!RFC9298}}
* CONNECT-IP {{!I-D.draft-ietf-masque-connect-ip}}
* Modern HTTP Proxies {{!I-D.draft-schwartz-modern-http-proxies}}

This specification provides a unified format for describing a collection of such access services, and a mechanism for reaching such services when the initial information contains only an HTTP origin.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Format

An access service collection is defined by a JSON dictionary containing keys specified in the corresponding registry ({{iana}}).  Inclusion of each key is OPTIONAL.  The corresponding media type is `application/access-services+json`.

The initially defined keys are "http", tcp", "dns", "udp", and "ip".  Each of these keys holds a JSON dictionary containing the key "template" with a value that is a URI template suitable for configuring a Modern HTTP proxy, Modern TCP Transport Proxy, DNS over HTTPS, CONNECT-UDP, or CONNECT-IP, respectively.  (Future keys might hold other JSON types.)

If the Access Description is for a general-purpose proxy, all proxy targets are presumed to be supported.  Otherwise, the supported targets must be understood from context.

## Examples

~~~JSON
{
  "http": {
    "template": "https://proxy.example.org/http{?target_uri}"
  },
  "tcp": {
    "template":
        "https://proxy.example.org/tcp{?target_host,tcp_port}"
  },
  "dns": {
    "template": "https://doh.example.com/dns-query{?dns}",
  },
  "udp": {
    "template":
        "https://proxy.example.org/masque{?target_host,target_port}"
  },
  "ip": {
    "template": "https://proxy.example.org/masque{?target,ip_proto}"
  }
}
~~~
{: title="A proxy with many supported services"}

# Client Authentication

General-purpose clients of this specification SHOULD also implement support for Popup Authentication {{!I-D.draft-schwartz-httpapi-popup-authentication}} when fetching the Access Description.  If Popup Authentication produced any Authorization or Cookie headers for the Access Description, the client MUST also apply those headers to any HTTP requests made using the included templates (subject to proxy-specific modifications, see {{Section 4.3 of !I-D.draft-schwartz-httpapi-popup-authentication}}).  These headers are applied even if the Access Description and the included services are on different HTTP origins.

This arrangement allows the use of Access Descriptions to describe access-controlled services, and avoids asking the user to authenticate separately with each service.

# Discovery from an Origin {#well-known}

In cases where the HTTP access service is identified only by an origin (e.g. when configured as a Secure Web Proxy), operators can publish an associated access service collection at the path "/.well-known/access-services", with the Content-Type "application/access-services+json".

Clients MAY fetch this Access Description and use the indicated services (in addition to any origin-scoped services) automatically.  Clients SHOULD use the description only while it is fresh according to its HTTP cache lifetime, refreshing it as needed.


# Security Considerations

TODO Security


# IANA Considerations {#iana}

IANA is requested to open a Specification Required registry entitled "HTTP Access Service Descriptors", with the following initial contents:

| Key   | Specification   |
|-------|-----------------|
| http  | (This document) |
| tcp   | (This document) |
| dns   | (This document) |
| udp   | (This document) |
| ip    | (This document) |

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
