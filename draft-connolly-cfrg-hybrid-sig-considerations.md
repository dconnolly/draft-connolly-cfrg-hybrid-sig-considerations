---
title: Hybrid Signature Considerations
abbrev: hybrid-sig-considerations
category: info

docname: draft-connolly-cfrg-hybrid-sig-considerations-latest
submissiontype: IRTF
number:
date:
consensus: true
v: 3
area: "IRTF"
workgroup: "Crypto Forum"
keyword:
 - post-quantum
 - hybrid
 - signatures

venue:
  group: "Crypto Forum"
  type: "Research Group"
  mail: "cfrg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/cfrg"
  github: "dconnolly/draft-connolly-cfrg-hybrid-sig-considerations"
  latest: "https://dconnolly.github.io/draft-connolly-cfrg-hybrid-sig-considerations/draft-connolly-cfrg-hybrid-sig-considerations.html"

author:
 -
    fullname: Deirdre Connolly
    organization: SandboxAQ
    email: durumcrustulum@gmail.com
 -
    fullname: Sophie Schmieg
    organization: Google
    email: sschmieg@google.com

normative:

  FIPS204:
    title: "Module-Lattice-Based Digital Signature Standard"
    date: August 13, 2024
    author:
      org: "National Institute of Standards and Technology (NIST)"
    target: https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf

  FIPS205:
    title: "Stateless Hash-Based Digital Signature Standard"
    date: August 13, 2024
    author:
      org: "National Institute of Standards and Technology (NIST)"
    target: https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.205.pdf


informative:

  HYBRIDSIG:
    target: https://eprint.iacr.org/2017/460
    title: Transitioning to a Quantum-Resistant Public Key Infrastructure
    author:
      -
        ins: N. Bindel
        name: Nina Bindel
      -
        ins: U. Herath
        name: Udyani Herath
      -
        ins: M. McKague
        name: Matthew McKague
      -
        ins: D. Stebila
        name: Douglas Stebila
    date: 2017-05

  HYBRIDSIGDESIGN:
    target: https://eprint.iacr.org/2023/423
    title: A Note on Hybrid Signature Schemes
    author:
    -
      ins: N. Bindel
      name: Nina Bindel
    -
      ins: B. Hale
      name: Britta Hale
    date: 2023-03

  I-D.ietf-tls-hybrid-design:

  I-D.ietf-pquip-pqt-hybrid-terminology:

  I-D.ietf-pquip-hybrid-signature-spectrums:

  I-D.ietf-lamps-pq-composite-sigs:


--- abstract

This document discusses how and when to use hybrid post-quantum/traditional
signatures, and when not to, including prehashing and key use.

--- middle

# Introduction {#intro}

This document discusses how and when to use hybrid post-quantum/traditional
signatures, and when not to, including prehashing and key use.

# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

# Specific Recommendations {#recommendations}

Working Groups SHOULD enable the use of ML-DSA signatures, as well as SLH-DSA
signatures and MAY enable the use of HashSLH-DSA signatures.

Working Groups SHOULD NOT include HashML-DSA as a signature option.

Working Groups SHOULD include non-hybrid options for signature schemes in
their protocols.

It is NOT RECOMMENDED to use hybrid signatures, except for rare
circumstances, and implementors MUST be warned to not reusing old key
material in a hybrid.

# Security Considerations {#security-considerations}

## Prehashing {#prehashing}

Prehashing allow the computation of a signature on two entities: The Message
Entity (ME) has access to the message and is not memory constrained, and a
signing entity (SE) that has access to the private key, but has limited
memory and bandwidth.  Prehashing allows the computation of a message
representative by the ME, which is then transfered to the SE to compute the
signature.

In some designs, the signature is computed in a streaming manner, i.e. the ME
first opens a stream to the SE, and then streams the entire message to the
SE, but the SE never has to buffer the entire message itself, and can operate
on the message stream only.  <!-- TODO: I basically just pulled that out of a -->
<!-- hat, we should look if there is academic literature on it. -->

## Security Considerations for ML-DSA {#ml-dsa}

ML-DSA allows for both streaming and prehashing messages. For prehashing,
this uses the comment on {{FIPS204}}, Algorithm 7, Line 6, and computes the
message representative `mu` in the ME as:

~~~
mu = SHAKE256(SHAKE256(pk, 64) || 0x00 || len(ctx) || ctx || m, 64)
~~~

where `pk` is the public key, `ctx` is the context string and `m` is the
message.

For streaming, the SE can accumulate the message by updating the SHAKE256
state, only having to keep track of the state.

ML-DSA supports both prehashed and streamed variants. These variants are interoperable
with the 'pure' ML-DSA defined in {{FIPS204}}, allowing a message signed with one
variant to be verified with the other. In contrast, the HashML-DSA variation defined
in {{FIPS204}} does not provide such property, it is superfluous and SHOULD NOT
be used to reduce interoperability difficulties.

## Security Considerations for SLH-DSA {#slh-dsa}

SLH-DSA, standardized in {{FIPS205}}, does not allow for prehashing or
streaming messages. The HashSLH-DSA variant defined in {{FIPS205}} MAY be
used to allow for prehashing and streaming. Alternatively, working groups can
design protocols in such a fashion that any message that has to be signed is
small enough to be transmitted over the network or be held in the memory of a
HSM.  If HashSLH-DSA is used, the identifier of the hash function used for the prehash
MUST be part of the public key. It is RECOMMENDED to use the same hash function for
the prehash as is used for the rest of SLH-DSA, but the hash function MUST
have collision resistance on par with the security level. (TODO maybe add a
list instead of leaving it like this)

In order to increase interoperability, it is RECOMMENDED to reduce the
overall number of variants, and only pick to support either SLH-DSA or
HashSLH-DSA.

SLH-DSA is proven to be secure as long as the hash function used as the
parameter of SLH-DSA is secure.  Since any signature scheme is only secure as
long as the hash function used with the scheme is secure, this means that in
scenarios where public keys cannot be revoked or expired, SLH-DSA can be used
to defend against the possibility of mathematical breakthroughs.

## Security Considerations for the use of Context {#using-context}

TODO

## Security Considerations for Hybrids {#hybrids}

The main argument for using hybrid signatures is that they continue to be
secure, even if one of the constituant schemes is broken.  While this is a
strong argument for encryption and KEMs, for signatures continuing to be
secure only matters if the public key cannot be changed.  Forging a signature
for a revoked public key is not a security vulnerability.

Systems SHOULD be designed to be able to recover from compromise, so they
usually have mechanism to revoke public keys, or only use short lived public
keys, which limit the damage of a compromised key. Breaking a signature
scheme compromises all keys using this scheme, but this break usually happens
fairly publicly.  If the protocol has the ability to switch to a new public
key, this can be used to mitigate the vulnerability posed by the broken
scheme. It is RECOMMENDED for protocols to include the algorithm used in the
public key, and not hardcode it or dynamically negogiate it, which would then
allow the migration to a new algorithm.

The benefit of this approach is that it works both in a prequantum and in a
post-quantum world, since it is agnostic of the type of breakage of the
algorithm, so designing protocols in this fashion also makes them robust for
the future.

### The Downsides of Hybrid Signatures {#hybrids-downsides}

There are two main downsides of hybrid signatures. Neither of them are
unavoidable, but both require conscious effort to avoid.

First, hybrid signatures technically allow for the continued use of the same
key material. However, if a key is used in both hybrid and non-hybrid cases,
this constitutes key reuse, and allows for stripping the PQC key and reusing
the signature as if it was classical only. The only way to avoid this problem
is to introduce a domain separator from a prefix-free set for both the
classical and the hybrid signature. Introducing it only for the hybrid
signature is not sufficient, since it would leave the classical signature
with the empty string as domain separator, but the empty string is a prefix
of every string, making the set of domain separators not prefix free. If all
clients can be updated to use the domain separated classical signature, they
can also be updated to use a new public key instead, and using a new key is
best practice, as it completely sidesteps this problem.  Hence protocol
designers MUST NOT allow the reuse of old key material in a hybrid.

The other downside of hybrid signatures is the combinatorial explosion that
occurs when the various classical schemes are combined with the various PQC
schemes.  It is RECOMMENDED to avoid this as much as possible by selecting
only very few possible combinations.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

TODO acknowledge.
