---
title: "BGP Signed Origin Delegation Attestation (SODA)"
abbrev: "BGP SODA"
category: std

docname: draft-jiang-idr-bgp-soda-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: "Routing"
workgroup: "Inter-Domain Routing"
# keyword:
#  - Inter-Domain Routing
#  - BGP
#  - ROV
# venue:
#   group: "Inter-Domain Routing"
#   type: Working Group
#   mail: WG@example.com
#   arch: https://example.com/WG
#   github: USER/REPO
#   latest: https://example.com/LATEST

author:
 -
    fullname: "Shenglin Jiang"
    organization: Zhongguancun Laboratory
    city: Beijing
    country: China
    email: "jiangshl@zgclab.edu.cn"
 -
    fullname: "Ke Xu"
    organization: Tsinghua University
    city: Beijing
    country: China
    email: "xuke@tsinghua.edu.cn"
 -
    fullname: "Zhuotao Liu"
    organization: Tsinghua University
    city: Beijing
    country: China
    email: "zhuotaoliu@tsinghua.edu.cn"
 -
    fullname: "Xiaoliang Wang"
    organization: Tsinghua University
    city: Beijing
    country: China
    email: "wangxiaoliang0623@foxmail.com"

normative:
  RFC2119:
  RFC4271:
  RFC6480:
  RFC6811:
  RFC8126:
  RFC8174:
  RFC8205:
  RFC8209:
  RFC8608:
  RFC9582:

informative:
  I-D.ietf-sidrops-aspa-verification:
  I-D.ietf-sidrops-rpki-prefixlist:
  I-D.ietf-sidrops-spl-verification:
  RFC8210:
  RFC8481:
  IMC2024:
    title: "Sublet Your Subnet: Inferring IP Leasing in the Wild"
    author:
      - name: Xiaohe Du
      - name: Romain Fontugne
      - name: Cecilia Testart
      - name: Alex C. Snoeren
      - name: kc claffy
    date: 2024
    seriesinfo:
      "In": "Proceedings of the ACM Internet Measurement Conference (IMC 2024)"
  NDSS2026:
    title: "Demystifying RPKI-Invalid Prefixes: Hidden Causes and Security Risks"
    author:
      - name: Weitong Li
      - name: Tao Wan
      - name: Taejoong Chung
    date: 2026
    seriesinfo:
      "In": "Proceedings of the Network and Distributed System Security Symposium (NDSS 2026)"
  Oakland26:
    title: "The Threat Landscape of IP Leasing in the RPKI Era"
    author:
      - name: Weitong Li
      - name: Yongzhe Xu
      - name: Taejoong Chung
    date: 2026-05
    target: https://weitongli.com/publications/papers/li-2026-hijack.pdf
    seriesinfo:
      "In": "Proceedings of the IEEE Symposium on Security and Privacy (Oakland'26), San Francisco, USA"

--- abstract

This document defines BGP Signed Origin Delegation Attestation (SODA),
an optional transitive BGP path attribute that carries a signed
authorization, issued by the holder of an IP prefix, delegating
origination of that prefix to a specified Autonomous System (AS) until a
stated expiry time. Using SODA, a validator performing Route Origin
Validation (ROV) can determine whether an ROV-Invalid result reflects an
authorized delegation rather than a route hijack.


--- middle

# Introduction


The Resource Public Key Infrastructure (RPKI) {{RFC6480}} and Route
Origin Authorizations (ROAs) {{RFC9582}} provide the foundation for
Route Origin Validation (ROV) {{RFC6811}}. A ROA authorizes a single
Autonomous System (AS) to originate routes for one or more specified IP
prefixes. When a BGP speaker performs ROV on a received UPDATE, a route
whose origin AS is not authorized by a covering ROA is classified as
Invalid.

ROV is effective at detecting unauthorized route origination. However,
an Invalid result does not necessarily indicate a route hijack. In
operational practice, a prefix holder may authorize another AS to
originate routes on its behalf without directly modifying the
corresponding ROA. Such arrangements arise in a variety of scenarios,
including address leasing, outsourced routing services, DDoS mitigation,
content delivery networks, and other forms of delegated route
origination {{IMC2024}}{{NDSS2026}}{{Oakland26}}.

Although a prefix holder could authorize a delegatee by creating an
additional ROA, doing so requires modifying the global RPKI repository
state and may be operationally undesirable for short-lived, dynamic, or
frequently changing delegations. Furthermore, ROAs are designed to
express route origination authorization rather than explicit delegation
relationships between a resource holder and a delegatee.

As a result, ROV alone cannot distinguish between a route hijack and a
route originated by an AS that has been legitimately authorized through
an out-of-band delegation arrangement. Both cases produce the same
ROV-Invalid outcome.

This document defines the BGP Signed Origin Delegation Attestation
(SODA), an optional transitive BGP path attribute named
ROUTE_DELEGATION_ATTEST. The attribute carries a cryptographically
signed delegation statement, issued by a prefix holder, authorizing a
specified AS to originate a prefix until a stated expiration time.

When standard ROV produces an Invalid result, a validator may evaluate
the SODA attribute to determine whether the route represents a
legitimate delegation rather than an unauthorized origination. SODA
therefore complements existing RPKI-based origin validation while
preserving the existing ROV trust model.


# Conventions and Definitions

{::boilerplate bcp14-tagged}



# Design Rationale

This section explains two design choices that are central to SODA and
that distinguish it from the obvious alternatives: carrying the
delegation authorization in band, as a BGP path attribute, rather than as
a published RPKI object; and introducing a new mechanism at all, rather
than expecting prefix holders to publish an additional ROA. These choices
also bound the situations in which SODA is the appropriate tool.

## Why an In-Band Attribute Rather Than a Published Object

The authorization that SODA conveys could in principle be expressed as a
new RPKI signed object, published to a repository and fetched by relying
parties, in the manner of ROAs {{RFC9582}}. SODA instead carries the
authorization in the BGP UPDATE that announces the delegated route. This
choice is motivated by the properties of the delegations that SODA
targets: short-lived, dynamic, or frequently changing arrangements such
as address leasing, outsourced routing, DDoS mitigation, and content
delivery {{IMC2024}}{{NDSS2026}}{{Oakland26}}.

A repository-published object becomes usable only after it has been
published at the issuer's publication point, fetched by relying parties
on their polling cycle, validated, and distributed to routers. The
resulting end-to-end delay is independent of BGP propagation and can be
substantial relative to the lifetime of an ephemeral delegation. An
in-band attribute, by contrast, travels with the route: the authorization
and the announcement arrive together and converge at BGP speed. For a
delegation whose intended lifetime is comparable to, or shorter than,
repository convergence latency, in-band delivery is the property that
makes the authorization useful at all.

In-band delivery also binds the authorization to a specific announcement.
A published object is global and standalone: it authorizes the delegatee
to originate the prefix toward every relying party, for the object's
entire validity period, independent of whether any corresponding route is
being announced. The SODA attribute exists only while the delegatee
announces the route, propagates only as far as the route propagates, and
is scoped to the prefix and the AS pair carried in the UPDATE. The
lifetime and reach of the authorization in the routing system therefore
track the lifetime and reach of the route itself, rather than persisting
in global state after the delegation has ended.

These benefits come at a cost, which this document does not minimize.
In-band delivery requires that the validating AS implement SODA in order
to obtain any benefit (see {{deployment}}), and it places signature
verification on, or adjacent to, the BGP processing path, with an
associated denial-of-service surface that is discussed in {{dos}}. SODA
further relies on the existing RPKI and on BGPsec
router certificates {{RFC8209}}; it introduces no new public key
infrastructure.

## Relationship to Publishing an Additional ROA

A prefix holder can already authorize a second origin AS by publishing an
additional ROA {{RFC9582}}. Where the delegation is long-lived and
stable, this is the appropriate tool, and operators SHOULD publish a ROA
rather than rely on SODA: a ROA is honored by every relying party that
performs ROV today, requires no new attribute support in the data path,
and is the mechanism the RPKI is designed to provide.

SODA targets the cases in which publishing a ROA is, by the prefix
holder's own choice, undesirable or insufficiently timely: delegations
that are too short-lived, too dynamic, or too numerous to be reflected in
the global repository without unacceptable churn or latency. In these
cases the relevant baseline is not "a ROA that already solves the
problem" but "no additional ROA, and therefore an ROV-Invalid route that
is discarded everywhere." Against that baseline, SODA is strictly
additive: at a SODA-aware validator it can recover a legitimately
delegated route, while at a validator that does not implement SODA the
route is treated exactly as it is today, namely as ROV-Invalid. SODA does
not weaken ROV and does not change the outcome for any route that a
prefix holder would have authorized with a ROA.

## Deployment Considerations {#deployment}

SODA provides benefit only between a delegator that issues attributes and
a validating AS that evaluates them; an AS that does not implement Phase 2
({{phase2}}) ignores the attribute and continues to treat the route as
ROV-Invalid. SODA therefore offers no network-wide benefit from unilateral
adoption, and its value to a given delegation grows with the number of
validators on the relevant paths that implement it. This is an inherent
property of an in-band, endpoint-evaluated mechanism and is the principal
cost of the design.

The corresponding failure mode is, however, benign. Because a SODA-Valid
outcome only ever causes a validator to accept a route that ROV alone
would have rejected, the absence of SODA support, the removal of the
attribute in transit (see {{removal}}), and the expiry of a delegation can
each only cause a delegated route to be treated as ROV-Invalid. None of
these conditions can cause an unauthorized route to be accepted. SODA thus
fails safe with respect to hijack in every partial-deployment scenario.

Operators deploying SODA should also account for the cost of evaluating
attributes attached to ROV-Invalid routes; this denial-of-service surface,
and the implementation measures that bound it, are discussed in {{dos}}.

# The ROUTE_DELEGATION_ATTEST Attribute

## Overview

ROUTE_DELEGATION_ATTEST is a new BGP path attribute. Its type code is
TBD (to be assigned by IANA). It is defined with the following flags:

* Optional (high-order bit of the Attribute Flags octet set to 1).
* Transitive (second-high-order bit set to 1).
* Extended Length (fourth-high-order bit set to 1) to accommodate the
  variable-length signature field.
* The Partial bit (third-high-order bit) follows standard BGP transitive
  attribute semantics per {{RFC4271}}.

The attribute carries a signed statement, issued by the delegator AS,
authorizing the delegatee AS to originate a specified prefix until a
stated expiration time.

## Attribute Format {#attr-format}

The value field of ROUTE_DELEGATION_ATTEST has the following format (all
multi-octet values are in network byte order):

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Version    |   Sig-Alg-ID  | Prefix-Length |   MaxLength   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Delegator-ASN                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Delegatee-ASN                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Delegation-Expiry                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Sig-Length          |      Prefix (octets 1-2)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Prefix (cont., variable, total 0-16 octets)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Signature (variable)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

All fixed-length fields occupy fixed octet offsets. The two
variable-length fields, Prefix and Signature, appear last: their lengths
are determined by Prefix-Length and Sig-Length, respectively, both of
which are carried in the fixed-length portion of the value. A parser
reads the fixed fields at known offsets, derives the Prefix length from
Prefix-Length, and then reads the Signature using Sig-Length.

Version (1 octet):
: The version of the SODA attribute format. This document defines
  Version 1 (value 0x01). Implementations MUST ignore (i.e., not use for
  validation) attributes with unknown Version values, and MUST forward
  them unchanged per transitive attribute semantics.

Sig-Alg-ID (1 octet):
: An identifier for the signature algorithm used. This document defines
  the value 0x01 for ECDSA with P-256 and SHA-256, the algorithm suite
  defined for BGPsec in {{RFC8608}}. Additional values may be registered
  as described in {{iana-algs}}. Validators that do not recognize the
  Sig-Alg-ID MUST treat the SODA attribute as invalid.

Prefix-Length (1 octet):
: The length, in bits, of the delegated prefix, using the encoding of
  {{RFC4271}} Section 4.3.

MaxLength (1 octet):
: The maximum prefix length of routes the delegatee is authorized to
  originate, following the semantics of the maxLength field in
  {{RFC9582}} Section 3. The value MUST be greater than or equal to
  Prefix-Length and MUST NOT exceed 32 for IPv4 or 128 for IPv6,
  according to the address family of the announced prefix (see Prefix
  below).

Delegator-ASN (4 octets):
: The 4-octet AS Number of the delegator, i.e., an AS belonging to the
  holder of the delegated prefix. The delegator signs this attribute with
  the private key of its BGPsec router certificate ({{RFC8209}}). The
  binding between the Delegator-ASN and the prefix, which establishes the
  delegator's authority to delegate origination, is verified as described
  in {{delegator-authority}}.

Delegatee-ASN (4 octets):
: The 4-octet AS Number of the delegatee, i.e., the AS authorized to
  originate routes for the prefix. This value MUST match the origin AS
  in the AS_PATH of the BGP UPDATE carrying this attribute.

Delegation-Expiry (4 octets):
: The expiry time of this delegation, expressed as a 32-bit unsigned
  integer representing seconds since the Unix epoch (January 1, 1970,
  00:00:00 UTC). This value indicates the end of the intended delegation
  period, independent of the validity period of any RPKI certificate or
  BGPsec router certificate. Validators MUST compare this field against
  the current time during SODA evaluation, as specified in {{phase2}}.

Sig-Length (2 octets):
: The length, in octets, of the Signature field. For Sig-Alg-ID 0x01
  (ECDSA P-256), the signature is encoded as a fixed 64-octet value, the
  concatenation of the two 32-octet integers r and s, as specified for
  BGPsec signatures in {{RFC8608}}.

Prefix (variable, 0-16 octets):
: The IP address prefix to which this delegation applies, encoded as the
  minimum number of octets necessary to represent the Prefix-Length most
  significant bits. Any trailing bits beyond Prefix-Length within the
  final octet MUST be set to zero so that the encoding (and therefore the
  signature) is canonical. IPv4 prefixes require at most 4 octets; IPv6
  prefixes require at most 16 octets. The address family is not carried in
  the attribute; it is taken from the address family of the announced NLRI
  against which the attribute is validated ({{phase2}}), and the Prefix
  MUST be interpreted, and the MaxLength bound applied, in that address
  family.

Signature (variable):
: The digital signature produced by the delegator using the private key
  corresponding to its BGPsec router certificate. The signed data is
  defined in {{signing}}.

## Signed Data {#signing}

The signature covers the following fields, concatenated in order without
any additional framing:

~~~
Signed-Data = Version            (1 octet)
            | Sig-Alg-ID         (1 octet)
            | Prefix-Length      (1 octet)
            | MaxLength          (1 octet)
            | Delegator-ASN      (4 octets)
            | Delegatee-ASN      (4 octets)
            | Delegation-Expiry  (4 octets)
            | Prefix             (variable, canonical, see above)
~~~

The Sig-Length and Signature fields are not covered by the signature. The
signed fields are exactly the value fields in wire order ({{attr-format}})
with Sig-Length and Signature removed. Including Version and Sig-Alg-ID in
the signed data binds the format version and the chosen algorithm to the
signature and prevents version or algorithm substitution.

# Operational Procedures

## Delegator Procedures

A delegator MUST be the holder of the delegated prefix in the RPKI, i.e.,
the resource holder to whom the prefix is allocated or sub-allocated. The
delegator's AS and the delegated prefix MUST therefore be covered by a
common RPKI resource certificate, so that the delegator's authority over
the prefix can be verified as described in {{delegator-authority}}. The
delegator MUST also hold a BGPsec router certificate for its AS, as
defined in {{RFC8209}}, whose private key it uses to sign the attribute.
Authority to delegate origination follows from holding the address space;
it is distinct from, and stronger than, mere authorization to originate
the prefix, which a ROA alone conveys.

To issue a ROUTE_DELEGATION_ATTEST attribute value for a delegation:

1. Construct the ROUTE_DELEGATION_ATTEST attribute value as specified in
   {{attr-format}}, with the appropriate Delegatee-ASN, Prefix,
   MaxLength, and Delegation-Expiry values.

2. Compute the signature over the Signed-Data as specified in
   {{signing}}, using the BGPsec router certificate private key.

3. Deliver the signed attribute value to the delegatee via an
   out-of-band channel. The out-of-band delivery protocol is outside the
   scope of this document.

The delegator SHOULD set Delegation-Expiry to a value no later than the
intended end of the delegation period. The delegator MAY issue a
replacement ROUTE_DELEGATION_ATTEST attribute value with an updated
Delegation-Expiry to extend an active delegation.


## Delegatee Procedures

A delegatee that has received a valid ROUTE_DELEGATION_ATTEST attribute
value from the delegator SHOULD attach the attribute to BGP UPDATE
messages for the delegated prefix.

The delegatee SHOULD cease advertising the ROUTE_DELEGATION_ATTEST
attribute after the Delegation-Expiry timestamp has passed, and SHOULD
withdraw the corresponding route unless a renewed attribute value has
been obtained.

A delegatee MUST NOT modify the value of a ROUTE_DELEGATION_ATTEST
attribute received from the delegator; the value MUST be included in BGP
UPDATE messages exactly as provided by the delegator.

A delegatee MUST NOT construct a ROUTE_DELEGATION_ATTEST attribute that
names its own AS as the Delegator-ASN unless it is itself the holder of
the prefix. An attribute self-issued without such authority will fail the
delegator-authority check in Step 2 of {{phase2}}.

Intermediate ASes that receive an UPDATE carrying a
ROUTE_DELEGATION_ATTEST attribute MUST NOT alter the attribute value. Per
standard transitive attribute semantics {{RFC4271}}, an AS that does not
recognize the attribute type code forwards it unchanged and sets the
Partial bit; setting the Partial bit does not affect SODA validation,
because the Signed-Data ({{signing}}) does not include the attribute
flags.

A BGP speaker that aggregates routes such that the announced prefix or the
origin AS of the resulting aggregate differs from the Prefix or the
Delegatee-ASN carried in a ROUTE_DELEGATION_ATTEST attribute MUST remove
the attribute from the aggregate, because the attestation no longer
corresponds to the announcement. If such an attribute is nonetheless
retained, it will fail validation (Step 5 or Step 6 of {{phase2}}) and the
aggregate will be treated as ROV-Invalid; SODA thus fails safe under
aggregation. Determination of the origin AS in the presence of an AS_SET
follows the rules of {{RFC6811}}.

# Validation Procedure

Validation of routes carrying ROUTE_DELEGATION_ATTEST follows a two-phase
procedure. Phase 1 is standard ROV; Phase 2 is the SODA fallback,
activated only if Phase 1 produces an Invalid result.

## Phase 1: Route Origin Validation

Perform ROV as specified in {{RFC6811}} and {{RFC9582}}:

1. Determine the effective origin AS from the AS_PATH attribute.

2. Query the local RPKI cache for ROAs covering the announced prefix.

3. If a matching ROA is found (prefix covered, origin AS matches, prefix
   length within maxLength), the result is ROV-Valid. The route is
   accepted; SODA evaluation is not performed.

4. If no ROA covers the announced prefix, the result is ROV-NotFound, and
   SODA evaluation is not performed: with no validated ROA, there is no
   RPKI anchor from which the delegator's authority over the prefix could
   be derived.

5. If a ROA covers the announced prefix but the origin AS does not match,
   the result is ROV-Invalid. Proceed to Phase 2 ({{phase2}}).

## Phase 2: SODA Fallback Validation {#phase2}

Phase 2 is entered only when Phase 1 produces ROV-Invalid. This indicates
that a ROA exists for the prefix (establishing an RPKI trust anchor) but
the origin AS does not match the ROA. Phase 2 determines whether this
mismatch is the result of a legitimate delegation. The checks below are
conjunctive: any failure yields the indicated result and terminates Phase
2. They MAY be performed in any order; for the performance implications of
ordering, see {{dos}}.

1. If the UPDATE does not carry a ROUTE_DELEGATION_ATTEST attribute, if
   the attribute is syntactically malformed, if it carries an
   unrecognized Version, or if it specifies an unrecognized Sig-Alg-ID,
   then SODA evaluation cannot be completed: the ROV-Invalid result
   stands and no further SODA processing is performed.

2. Verify, using the validated RPKI certificate tree, that the
   Delegator-ASN and the announced prefix are covered by a common RPKI
   resource certificate, i.e., that a single resource holder holds both
   the Delegator-ASN (as an AS resource) and the announced prefix (as an
   IP resource). If no such common certificate exists, the result is
   SODA-Invalid. This step anchors the delegation to address-space
   holdership: it establishes that the delegator holds the prefix and thus
   has authority to delegate its origination, rather than merely
   permission to originate it (which a ROA conveys and which is not
   sufficient; see {{delegator-authority}}). This check requires access to
   RPKI certificate resource information, which is available to a relying
   party with repository access (the same access used in Step 3).

3. Using the Delegator-ASN, retrieve the corresponding BGPsec router
   certificate from the RPKI repository, as specified in {{RFC8209}}, and
   extract its public key. If no valid BGPsec router certificate is found
   for the Delegator-ASN, the result is SODA-Invalid.

4. Using the retrieved public key and the algorithm identified by
   Sig-Alg-ID, verify the Signature over the Signed-Data defined in
   {{signing}}. If signature verification fails, the result is
   SODA-Invalid.

5. Verify that the announced prefix is consistent with the delegated
   scope carried in the attribute: the most significant Prefix-Length
   bits of the announced prefix MUST equal the Prefix field, and the
   length of the announced prefix MUST be greater than or equal to
   Prefix-Length and less than or equal to MaxLength. If the announced
   prefix falls outside this scope, the result is SODA-Invalid. This step
   ensures that a delegation signed for one prefix cannot be used to
   authorize a different prefix.

6. Verify that the Delegatee-ASN field equals the effective origin AS
   determined from the AS_PATH attribute of the BGP UPDATE. If the values
   do not match, the result is SODA-Invalid.

7. Compare the Delegation-Expiry value against the current time at the
   moment of evaluation. If the current time is greater than or equal to
   Delegation-Expiry, the result is SODA-Expired. Otherwise (the current
   time is strictly less than Delegation-Expiry), the result is
   SODA-Valid.

## Local Policy {#outcome}

This document defines the SODA validation outcomes SODA-Valid,
SODA-Invalid, and SODA-Expired, which operators can incorporate into
local routing policy. Consistent with {{RFC8481}}, a router applies no
SODA-derived policy to a route except as configured by the operator.
Implementations SHOULD make the following behaviors configurable:

* Accept routes with a SODA-Valid outcome (RECOMMENDED for operators with
  delegatee relationships).

* Log and alert on SODA-Expired outcomes to trigger delegator
  notification.

* Apply a more restrictive local policy to routes with a SODA-Invalid
  outcome than to routes with an ordinary ROV-Invalid outcome, since the
  presence of a ROUTE_DELEGATION_ATTEST attribute that fails verification
  may indicate an active attempt to misuse the delegation mechanism.

Because BGP does not provide a mechanism for timed route withdrawal,
validators SHOULD periodically re-evaluate locally cached routes against
the Delegation-Expiry timestamps of any stored ROUTE_DELEGATION_ATTEST
attributes. The appropriate re-evaluation interval is a matter of local
policy; implementations SHOULD support configuration of this interval.

# Relationship to Existing Mechanisms

SODA is designed to complement, not replace, existing BGP security
mechanisms. The following summarizes the relationship between SODA and
related standards:

ROV and ROA ({{RFC6811}}, {{RFC9582}}):
: Phase 1 of SODA validation performs standard ROV as defined in
  {{RFC6811}}. A valid ROA covering the announced prefix is a necessary
  precondition for SODA evaluation; SODA does not substitute for ROV. The
  NotFound result explicitly excludes SODA evaluation, preserving the
  integrity of the RPKI trust chain.

BGPsec ({{RFC8205}}):
: SODA relies on BGPsec router certificates defined in {{RFC8209}} for
  signature validation. No new PKI is required. SODA and BGPsec address
  orthogonal properties (origin delegation versus path integrity) and may
  coexist in the same UPDATE message.

Signed Prefix Lists (SPL) ({{I-D.ietf-sidrops-rpki-prefixlist}}, {{I-D.ietf-sidrops-spl-verification}}):
: An SPL is an RPKI object, published to the repository, in which an AS
  lists the complete set of prefixes it may originate. As the SPL profile
  notes, an SPL is self-asserted by the AS holder and carries no authority
  from any prefix holder; that authority is conveyed separately by a ROA.
  SODA differs in both trust root and delivery: a SODA attestation is
  signed by the holder of the prefix (the delegator), is bound to a
  specific announcement, and is carried in band rather than published as a
  repository object. SODA is therefore a prefix-holder authorization for a
  third party to originate, whereas an SPL is a self-description by an
  origin AS of its own intended originations.

# Security Considerations

## Scope of Delegator Authority {#delegator-authority}

A ROA expresses that an AS may originate a prefix; it does not express
that the AS may further delegate origination of that prefix to a third
party. If SODA anchored delegation authority in a ROA's origin AS, then
any AS that a prefix holder had authorized to originate the prefix (for
example, a transit provider or a previous delegatee) could in turn issue
SODA attestations naming arbitrary further delegatees, an escalation the
prefix holder may not have intended.

For this reason, SODA anchors delegation authority in address-space
holdership rather than in origination authorization. Step 2 of {{phase2}}
requires the Delegator-ASN and the announced prefix to be covered by a
common RPKI resource certificate, establishing that a single resource
holder holds both the AS and the prefix and therefore that the delegator
is entitled to delegate the prefix's origination.

## Stale Route Window

BGP does not natively support timed route withdrawal. When a
Delegation-Expiry timestamp passes, the route remains in the RIB as
SODA-Expired until a BGP UPDATE or withdrawal is received or until the
validator performs a periodic re-evaluation sweep. Operators SHOULD
configure validators to perform periodic re-evaluation as described in
{{outcome}} and SHOULD configure delegatees to withdraw delegated routes
promptly upon expiry.

## BGPsec Key Compromise

If a delegator's BGPsec router certificate private key is compromised, an
attacker can forge SODA attributes for any prefix held by the same
resource holder as the delegator's AS, i.e., any prefix covered by a
common RPKI resource certificate with that AS (see
{{delegator-authority}}). The delegator SHOULD revoke the compromised
BGPsec router certificate immediately, which will cause all outstanding
SODA attribute values signed by that key to fail Step 3 of {{phase2}}.
Certificate revocation follows BGPsec key rollover procedures as defined
in {{RFC8209}}.

## Replay of Valid SODA Attributes

A valid ROUTE_DELEGATION_ATTEST attribute may be replayed by an attacker
until the Delegation-Expiry timestamp is reached. The prefix-consistency
check in Step 5 of {{phase2}} confines any such replay to the exact
prefix and length range named in the signed attribute; it cannot be used
to authorize a different prefix.

This document relies on bounded delegation lifetimes to limit replay
exposure. Operators are encouraged to use relatively short delegation
lifetimes and to renew delegations as necessary.

## Removal of the Attribute {#removal}

Because ROUTE_DELEGATION_ATTEST is an optional, transitive attribute, an
on-path adversary on a BGP session can remove it from an UPDATE. Removal
does not enable a route hijack: without the attribute, the route reverts
to an ordinary ROV-Invalid result and is rejected by validators applying
the RECOMMENDED policy in {{outcome}}. Removal can, however, cause a
legitimately delegated route to be rejected. SODA therefore fails safe
against hijack, but does not protect the availability of a delegated
route against an adversary able to strip the attribute in transit.

## Computational Cost and Denial of Service {#dos}

Phase 2 of validation ({{phase2}}) performs RPKI certificate-tree lookups
and a signature verification. Because Phase 2 is reached only for
ROV-Invalid routes, an adversary can attach a malformed or unverifiable
ROUTE_DELEGATION_ATTEST attribute to a large number of announcements that
are already ROV-Invalid, attempting to induce expensive lookups and
verifications at validators.

To bound this cost, and because the Phase 2 checks may be performed in any
order, implementations SHOULD perform the inexpensive checks first: the
attribute presence, well-formedness, Version, and Sig-Alg-ID checks of
Step 1; the prefix-consistency check of Step 5; and the Delegatee-ASN
match of Step 6. Only if those succeed should an implementation perform
the RPKI lookups of Steps 2 and 3 and the signature verification of Step
4. Implementations SHOULD cache Phase 2 results keyed on the attribute
value, so that repeated announcements carrying the same attribute are not
re-verified, and SHOULD be able to rate-limit Phase 2 processing,
including on a per-peer basis. These measures keep the additional work
introduced by SODA bounded even under a flood of crafted attributes.

## Path Validation

SODA provides attestation of route origin delegation only. It does not
validate the AS_PATH beyond checking that the origin AS matches the
Delegatee-ASN. SODA is orthogonal to and compatible with BGPsec
{{RFC8205}} and ASPA {{I-D.ietf-sidrops-aspa-verification}}, which address
path validity. Operators requiring end-to-end path security SHOULD deploy
SODA in conjunction with BGPsec or ASPA.

# IANA Considerations

## BGP Path Attribute Type Code

IANA is requested to assign a new type code in the "BGP Path Attributes"
registry for the ROUTE_DELEGATION_ATTEST attribute defined in this
document:

| Value | Code                    | Reference     |
| ----- | ----------------------- | ------------- |
| TBD   | ROUTE_DELEGATION_ATTEST | This document |

## SODA Signature Algorithm Identifiers {#iana-algs}

IANA is requested to create a new registry, "SODA Signature Algorithm
Identifiers", to record the one-octet Sig-Alg-ID values used by the
ROUTE_DELEGATION_ATTEST attribute. The registration policy is IETF Review
{{RFC8126}}.

The initial contents of the registry are:

| Value     | Algorithm                | Reference     |
| --------- | ------------------------ | ------------- |
| 0x00      | Reserved                 | This document |
| 0x01      | ECDSA P-256 with SHA-256 | RFC 8608      |
| 0x02-0xFF | Unassigned               |               |

--- back

# Acknowledgements
{:numbered="false"}

The authors thank the researchers at CAIDA, Virginia Tech, and CableLabs
whose measurement studies provided the empirical foundation for this
work.
