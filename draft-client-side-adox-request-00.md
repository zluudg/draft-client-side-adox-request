---
title: "Client-side Request for Encrypted Authoritative DNS Queries"
abbrev: "Client-side ADoX Request"
docname: draft-client-side-adox-request-00
date: {DATE}
category: std

ipr: trust200902
area: Internet
workgroup: DNSOP Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
  ins: L. Fernandez
  name: Leon Fernandez
  organization: The Swedish Internet Foundation
  country: Sweden
  email: leon.fernandez@internetstiftelsen.se

normative:

informative:

--- abstract

This document describes an EDNS0 option that a client can attach to a
DNS query to signal to a recursive resolver that it should prefer
encrypted transports while resolving the query. The option can also
be used on the return path by the recursor to signal whether encryption
was achievable or not.


TO BE REMOVED: This document is being collaborated on in Github at:
[https://github.com/zluudg/draft-client-side-adox-request](https://github.com/zluudg/draft-client-side-adox-request).
The most recent working version of the document, open issues, etc, should all be
available there.  The authors (gratefully) accept pull requests.

--- middle

# Introduction TODO(refs)
A DNS client can choose what protocol to use when it sends a query to
a resolver. There are several privacy-friendly options that the client
can choose between such as HTTPS, QUIC and TLS. However, the choice
only affects the first hop and the query will typically be sent in
cleartext by the resolver during the resolution process. In order to
preserve the privacy of the client beyond the first hop, the client
should be able to signal a desire that the resolver uses encrypted
transports during the resolution process. Furthermore, the client
should be informed of how the resolution was carried out, in case it
would like to discard data that was not received over an encrypted
transport.

This document defines an EDNS0 {{!RFC6891}} option that allows a client
to signal a preference for encrypted authoritative DNS (ADoX). By
signalling a preference, the client can urge a recursive resolver to
discover possibilities for encrypted transport. Means of discovery
include, but are not limited to, {{!RFC9539}}, {{?I-D.johani-dnsop-transport-signaling}}
and {{?I-D.wesplaap-deleg-transport}}. The signal from the client can
also prevent the recursive resolver from leaking queries that would
otherwise have been sent in cleartext. The same EDNS0 option can also
be used on the return path so that the recursive resolver can give an
indication to the client about what level of privacy was achieved for
a particular query.

# Prior Art
TODO(QNAME minimization, how much is okay to leak? TLD? None at all?
define a value that requests qname minimization?)

**Note to the RFC Editor**: Please remove this entire section before publication.

# Terminology

Client: A stub resolver, typically part of an end-users operating system
    or web browser.

Recursive Resolver/Recursor: A resolver that resolves a query on behalf
    of a client. This is done by first querying the root of the DNS tree
    and, if necessary, following the chain of delegations with more
    queries until an authoritative server is found.

ADoX: Authoritative DNS-over-X, where X implies some arbitrary encrypted
    transport protocol.

# The EDNS0 PRIVACY Option

The EDNS0 PRIVACY option is structured as follows:

~~~
                                               1   1   1   1   1   1
       0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 0:  |                            OPTION-CODE                        |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 2:  |                           OPTION-LENGTH                       |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 3:  |           PRIVACY             |
     +---+---+---+---+---+---+---+---+
~~~

Field definition details:

OPTION-CODE: The EDNS0 option code for client-privacy-request, defined
    as TBD.

OPTION-LENGTH: The option length SHOULD have a value of exactly 1 if
    this option is present. Consumers of this signal SHOULD NOT act on
    malformed options.

PRIVACY: For queries, this value encodes the client's desired level of
    privacy. For responses, this value encodes the level of privacy
    that the resolver was able to achieve. Currently defined values are
    listed in {{privacy-values}}.

# Privacy Values {#privacy-values}

| Value | Context  | Mnemonic      |
|-------|----------|---------------|
| 0     | Query    | NONE          |
| 0     | Response | CLEARTEXT     |
| 1     | Query    | OPPORTUNISTIC |
| 1     | Response | ENCRYPTED     |
| 2     | Query    | STRICT        |
| 2     | Response | CACHED        |

## Interpretations of Privacy Values in a query context

### NONE
Same effect as having to client signal at all. No privacy requested by
the client that is sending the query.

### OPPORTUNISTIC
The client is requesting a preference for resolving the query using
encrypted transports. However, fallback to unencrypted transport is
acceptable.

### STRICT
The client is requesting a strict preference for resolving the query
using encrypted transports. Leaking information about the query over
unencrypted transports is not acceptable. 

## Interpretations of Privacy Values in a response context

### CLEARTEXT
The answer was fetched over an unencrypted connection.

### ENCRYPTED
The answer was fetched only over encrypted transports.

### CACHED TODO(Enough to say this? Maybe we want to know if it was validated or received of encr transport)
The answer was fetched from the resolver's cache and no queries had to
be sent out.

# Protocol Description
A client adds the PRIVACY EDNS0 option to a recursive query before
sending it. A client SHOULD NOT add the PRIVACY option to recursive
queries being sent in cleartext as this would be counter to what the
client is trying to achieve.

Given that the client sent its query using an encrypted transport, the
recursive resolver will try to honor the PRIVACY option. If it is NONE,
it means the client does not care about what transports are used to
resolve the query and the recursor may resolve it in any way deemed
suitable. If the option is OPPORTUNISTIC, the recursor MUST attempt to
send any iterative queries over an encrypted transport. However, the
recursor is allowed to fall back to an unencrypted transport after a 
failed attempt. If the option is STRICT the recursor MUST NOT perform
such a fallback at any point during the resolution process. Using
contents from the cache is always okay as this does not leak any
information about the client's query.

## Setting a Response PRIVACY Value
A recursive resolver should only set a EDNS0 PRIVACY option if the
option was present in the query. If the data was readily available in
the recursor's cache, it sets a value of CACHED. If at any point
during the resolution process, a query was sent unencrypted (either
due to falling back or to the client query having a value of NONE), the
resolver MUST set the value to CLEARTEXT. If the resolver solely used
encrypted transports and data from its cache to retrive the answer, it
responds with a value of ENCRYPTED. What transport cached data arrived
over does not matter in this case.

# Security Considerations
Enables/simplifies cache probing through the CACHED option. Not a big
deal, TTL can be used for that anyway.

TODO(Discussion of passive vs. active attacks. A STRICT setting is still
vulnerable to active attacks, but the name STRICT might imply that it
isn't for some people.)

# Operational Considerations

# IANA Considerations

TODO

~~~
   +-------+--------------------------+----------+----------------------+
   | Value | Name                     | Status   | Reference            |
   +-------+--------------------------+----------+----------------------+
   | TBD   | client-privacy-request   | Standard | ( This document )    |
   +-------+--------------------------+----------+----------------------+
~~~

**Note to the RFC Editor**: In this section, please replace
occurrences of "(This document)" with a proper reference.

# Implementation Status

**Note to the RFC Editor**: Please remove this entire section before publication.

The TDNS Framework of experimental DNS servers developed and maintained by the
Swedish Internet Foundation implements this draft (see [https://github.com/johanix/tdns](https://github.com/johanix/tdns)). TDNS has support for both the authoritative nameserver and
recursive nameserver parts of the draft.

# Acknowledgments**

--- back

# Change History (to be removed before publication)

> Initial public draft
