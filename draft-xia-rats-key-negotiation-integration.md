---
v: 3

title: "Integration of Remote Attestation with Key Negotiation and Distribution"
abbrev: "RATS Key Negotiation Integration"
docname: draft-xia-rats-key-negotiation-integration-latest
category: std
consensus: true
submissiontype: IETF

ipr: trust200902
area: "Security"
workgroup: "Remote ATtestation ProcedureS"
keyword: [ RATS, attestation, key negotiation ]

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

venue:
  group: "Remote ATtestation ProcedureS"
  type: "Working Group"
  mail: rats@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/rats/
  github: ietf-rats/draft-xia-rats-key-negotiation-integration

author:
  - name: Liang Xia
    org: Huawei Technologies
    email: frank.xialiang@huawei.com
  - name: Weiyu Jiang
    org: Huawei Technologies
    email: jiangweiyu1@huawei.com
  - name: Muhammad Usama Sardar
    org: TU Dresden
    email: muhammad_usama.sardar@tu-dresden.de
  - name: Henk Birkholz
    org: Fraunhofer SIT
    email: henk.birkholz@ietf.contact
  - name: Jun Zhang
    org: Huawei Technologies France S.A.S.U.
    email: junzhang1@huawei.com
  - name: Houda Labiod
    org: Huawei Technologies France S.A.S.U.
    email: houda.labiod@huawei.com

contributor:

normative:

informative:

    I-D.fossati-tls-attestation:
    I-D.fossati-tls-exported-attestation:
    I-D.ietf-lamps-csr-attestation:

--- abstract

This draft proposes a lightweight security enhancement scheme based on remote attestation—key negotiation integrated into remote attestation. Organically integrating the main steps of end-to-end key negotiation into the remote attestation process may provide more security and flexibility.

--- middle

# Introduction {#intro}

Remote attestation is a security mechanism based on trusted hardware (e.g., TPM, TEE), allowing remote verifiers to cryptographically verify the integrity of the target device's software configuration, hardware state, and runtime environment.
Hence, remote attestation can effectively prove the overall security state of the endpoint.
Secure channel protocols (e.g., TLS, QUIC, IPSec, SSH) establish end-to-end (E2E) secure channels based on the authentication of the endpoint's legitimate identity and secure key negotiation, ensuring the security of network communication.
By organically combining remote attestation protocols with secure channel protocols and establishing cryptographic binding between them, it is possible to achieve a logical binding of endpoint security and network security, ensuring dual verification and protection of the identity and state of the endpoint in secure connections. Attested TLS {{I-D.fossati-tls-attestation}} {{I-D.fossati-tls-exported-attestation}} is currently an important related work in the industry, and other similar works include binding remote attestation with credential issuance (e.g., certificates {{I-D.ietf-lamps-csr-attestation}}, OAuth tokens, etc.) to achieve security enhancement.

However, in some scenarios, the above binding may not be possible.
For example:

* Scenario 1: When tenants in a public cloud/compute cluster deploy workloads, they need to first verify their runtime environment's security through remote attestation before requesting Key Management Service (KMS) to assign application-layer data keys to the Virtual Machine (VM)/compute node where the workload resides.
At this point, these keys are used for application-layer data encryption, such as file encryption or disk encryption, and are not used for any secure channel protocols.
For example, to protect model parameters from leakage and tampering, model weights and other parameters need to be encrypted before model loading and transmitted to a Trusted Execution Environment (TEE). At this time, it is necessary to ensure that only the TEE has the key and decrypts it.
* Scenario 2: The end user/client accesses the online TEE computing environment, submits his data for business processing or large model inference, and needs to ensure the security of the entire computing environment through remote attestation before establishing a secure connection.
At this point, there are multiple options for the secure channel protocol that can be established, such as TLS, IPSec, QUIC, OHTTP, etc.
The user may only need to complete E2E key negotiation based on remote attestation.
As for which secure protocol or application layer encryption the negotiated key is used for, and how it is used, there can be various implementation methods.

In summary, considering the diversity of remote attestation application scenarios and the limitations or complexity of combining with security protocols, this draft proposes a lightweight security enhancement scheme based on remote attestation—key negotiation integrated into remote attestation. By organically integrating the key steps of E2E key negotiation into the remote attestation process, the following can be achieved:

* The key distribution of KMS or E2E key negotiation can be automatically completed based on remote attestation, improving the security of key negotiation;
* The keys negotiated automatically can be flexibly applied in various ways, whether for secure protocols or application layer encryption;
* Compared to the complete and systematic implementation of Attested TLS, a more lightweight implementation can be provided.

## Requirements Notation

{::boilerplate bcp14-tagged}

# Integration Scheme
The current specification is based on passport model of RATS. Future versions of specification will include the background-check model. We present two integration schemes:

## Key Distribution Based on KMS Integrated with Remote Attestation

[questions]: <> (Q. Which key is being distributed here? Q. What is trust root? Q. Secure channel between whom? Q. Attester can request SK etc. But how can Attester provide?)

The KMS mechanism on public cloud networks includes trust root and secure channels, full lifecycle management of keys (including key generation, storage, rotation, and destruction), hierarchical encryption architecture (such as Envelope Encryption), and access control mechanisms.
KMS enables applications to generate/obtain their own application-layer encryption symmetric/asymmetric keys and distribute these keys between the required applications.
Furthermore, the method of integrating key distribution into the remote attestation interaction process is shown in the following diagram:

~~~
        .------------.
        |            | Compare Evidence
        |  Verifier  | against appraisal policy
        |            |
        '--------+---'
            ^    |
1. Evidence |    | 2. Attestation
            |    | Result
            |    v
        .---+--------.                .-------------.
        |            +--------------->|   Relying   | 4. Compare Attestation
        |  Attester  |3. Attestation  |    Party    | Result against
        |            |   Result       |    & KMS    | appraisal policy
        '------------' + SK/ASK/ESK   '-------------' + Distribute Keys

SK：Symmetric Key
ASK：Asymmetric Key
ESK：Encrypted Symmetric Key
~~~
{: #fig-rats-key-negotiation-integration-kms title="Key Distribution Based on KMS Integrated with Remote Attestation"}

In the standard remote attestation process described above, the Attester can generate and provide/request the Attester's application layer SK/ASK/ESK keys in the result returned to the KMS.
The KMS can generate or distribute these keys accordingly. During key rotation, the KMS can proactively trigger the above process to complete the update and rotation of the new and old keys.

## Integrating E2E Key Negotiation Into Remote Attestation

[questions2]: <> (Q. Why Attester should include pubS in AR rather than Evidence?)

The current main implementation mechanism for E2E key negotiation is DHE and ECDHE. Taking ECDHE as an example, the method of integrating it into remote attestation is shown in the following figure:

~~~
       .------------.
       |            | Compare Evidence
       |  Verifier  | against appraisal policy
       |            |
       '--------+---'
           ^    |
3 Evidence |    | 4. Attestation
           |    | Result
           |    v     1. Attestation
      .---+--------. Request + pubC.-------------.
      |  Attester  <-------------->|   Relying   | 6. Compare Attestation
      |  & Server  | 5.Attestation |    Party    | Result against
      |            | Result        |    & Client | appraisal policy
      '------------' + pubS        '-------------' + SK = privC * pubS
    2. SK = privS * pubC

SK：Symmetric Key
(privC,pubC): Client's ECDHE key pair
(privS,pubS): Server's ECDHE key pair
~~~
{: #fig-rats-key-negotiation-integration-e2e title="Integrating E2E Key Negotiation Into Remote Attestation"}

In the standard remote attestation process described above, the Client includes the public key pubC from its dynamically generated ECDHE key pair (privC, pubC) in the remote attestation request message, while retaining its private key privC.
Upon receiving pubC, the Server can compute the symmetric key SK using its private key privS from its dynamically generated ECDHE key pair (privS, pubS).
After completing the remote attestation with the Verifier, the Server includes its pubS in the Attestation Result returned to the Client.
Once the Client verifies the Attestation Result, it can compute the symmetric key SK using pubS and its own private key privC, thereby completing both the remote attestation and ECDHE key agreement.

# Use of Negotiated Keys in Different Security Protocols

Through the above process, key negotiation/distribution is completed during the remote attestation process, and is synchronized based on the results of the remote attestation.
If the negotiated key is used for application layer encryption, its specific usage is strongly related to the application and can be very flexible.
When the key is used for the security protocols, such as TLS, IPSec, etc., there are at least the following binding methods:

* TLS: The negotiated key can be used as a pre-shared key for subsequent TLS handshakes; the key can also be used as an externally imported shared key to participate in TLS Hybrid key exchange {{?I-D.ietf-tls-hybrid-design}};
* IPSec: The negotiated key can be used as a pre-shared key for subsequent IKEv2 handshakes; or the key can be directly used as a session key for data plane encryption and integrity protection in IPSec ESP; or the key can also be used as Post-quantum Preshared Keys (PPKs) {{!RFC8784}} to achieve binding with the IPSec protocol.

# Remote Attestation Protocol and Message Extensions

This section describes how to extend RATS protocol and message to incorporate key negotiation into the remote attestation process.

TBD

# Security Considerations

Risk of relay attacks needs to be evaluated in the design.

TBD

# Privacy Considerations

TBD

# IANA Considerations

TBD

--- back
