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
* Scenario 3: A company intends to deploy its private large model or file system to a public cloud platform. However, for security reasons, it first uploads the encrypted objects to the cloud platform. Subsequently, it needs to conduct a security check on the TEE (Trusted Execution Environment) within the cloud platform's computing environment using remote attestation. Once the security check is passed, the decryption key is sent to decrypt the uploaded objects.

It is important to note that between the remote attestation and the key negotiation, a key binding mechanism between the  attestation result and the attester's identity must be established to prevent diversion attack.

In summary, considering the diversity of remote attestation application scenarios and the limitations or complexity of combining with security protocols, this draft proposes a lightweight security enhancement scheme based on remote attestation—key negotiation integrated into remote attestation. By organically integrating the key steps of E2E key negotiation into the remote attestation process, the following can be achieved:

* The key distribution of KMS or E2E key negotiation can be automatically completed based on remote attestation, improving the security of key negotiation;
* The keys negotiated automatically can be flexibly applied in various ways, whether for secure protocols or application layer encryption;
* Compared to the complete and systematic implementation of Attested TLS, a more lightweight implementation can be provided.

## Requirements Notation

{::boilerplate bcp14-tagged}

# Integration Scheme
The current specification is based on passport model of RATS. Future versions of specification will include the background-check model. We present two integration schemes:

## Public Cloud KMS Key Distribution Integrated with Remote Attestation

The KMS mechanism on public cloud networks includes the root of trust, secure channels between Attester and KMS, full lifecycle management of keys (including key generation, storage, rotation, and destruction), hierarchical encryption architecture (such as Envelope Encryption), and access control mechanisms. Specifically:

* The basic process for symmetric key distribution and usage is as follows: When an application needs to encrypt data and share it, it requests the KMS to distribute the DEK (Data Encryption Key) and EDEK (Encrypted DEK, encrypted with the CMK) to the application via an API. The application then encrypts the data using the DEK, deletes the DEK from memory, and sends the encrypted ciphertext along with the EDEK to the receiving application. The receiving application subsequently requests the KMS to decrypt the EDEK via an API, retrieves the DEK to decrypt the data, and then deletes the DEK from memory.
* The basic process of asymmetric key distribution and usage is as follows: When an application needs to encrypt data, it requests the KMS to generate a public-private key pair via an API, and then uses the obtained public key to encrypt the data. The encrypted ciphertext is subsequently sent to the receiving application. The receiving application calls the KMS's decryption API, and the KMS internally uses the corresponding private key to decrypt the data, returning the plaintext to the receiving application. The private key never leaves the KMS. When an application needs to perform digital signing on data, it requests the KMS to generate a public-private key pair via an API, then distributes the public key to all parties that need to verify the signature. The application then requests the KMS to sign the data using the corresponding private key via an API, and sends the signature result along with the data to the receiving application. The receiving application uses the public key to verify the signature.

In summary, KMS supports providing the applications the required deliverables, which are not limited to the application-layer encryption/authentication symmetric/asymmetric keys, or the decrypted plaintext, or the calculated signatures. For simplicity, keys are used here as a general term for all these deliverables.

Furthermore, the method of integrating key distribution into the remote attestation interaction Passport Model process is shown in the following diagram:

~~~
            .------------.
            |            | 2. Compare Evidence
            |  Verifier  | against appraisal policy
            |            |
            '--------+---'
                ^    |
  1. Evidence + |    | 3. Attestation
  Attester's    |    | Result + Attester's identity
  identity      |    v
            .---+--------.                .-------------.
            |            +--------------->|   Relying   | 5. Compare Attestation
            |  Attester  |4. Attestation  |    Party    | Result against
            |            |   Result +     |    & KMS    | appraisal policy
            '------------' Attester's     '-------------' + Distribute Keys
                           identity                         with Attester's public key

~~~
{: #fig-rats-key-negotiation-integration-cloud-kms title="Public Cloud KMS Key Distribution Integrated with Remote Attestation Passport Model Interaction"}

In the standard remote attestation process described above, the Attester can request the Attester's application layer keys after it provide its attestation result to the KMS. By including the attester's identity (raw public key or certificate) in the messages throughout the remote attestation process and having the attester (using its attestation evidence signing key) and verifier (using its attestation result signing key) endorse and sign it, a key binding mechanism between the attester's attestation result and its identity is implemented. Subsequently, the KMS can use the identity's public key for key distribution, ensuring that the keys are distributed to the correct attester, thereby eliminating diversion attack. During key rotation, the KMS can proactively trigger the above process to complete the update and rotation of the new and old keys.

## Integrating E2E Key Negotiation Into Remote Attestation

The current main implementation mechanism for E2E key negotiation is DHE and ECDHE. Taking ECDHE as an example, the method of integrating it into remote attestation passport Model process is shown in the following figure:

~~~
           .------------.
           |            | 4. Compare Evidence
           |  Verifier  | against appraisal policy
           |            |
           '--------+---'
 3. Evidence + ^    |
    pubS +     |    |5. Attestation
    Attester's |    | Result + pubS + Attester's identity
    identity   |    v     1. Attestation
          .---+--------. Request + pubC.-------------.
          |  Attester  <-------------->|   Relying   | 7. Compare Attestation
          |  & Server  | 6.Attestation |    Party    | Result against
          |            | Result        |    & Client | appraisal policy
          '------------' + pubS        '-------------' + SK = privC * pubS
  2. SK = privS * pubC   + Attester's                  + store the mapping of SK
                           identity                      with Attester's identity
SK：Symmetric Key
(privC,pubC): Client's ECDHE key pair
(privS,pubS): Server's ECDHE key pair
~~~
{: #fig-rats-key-negotiation-integration-e2e title="Integrating E2E Key Negotiation Into Remote Attestation Passport Model Interaction"}

In the standard remote attestation process described above, the Client includes the public key pubC from its dynamically generated ECDHE key pair (privC, pubC) in the remote attestation request message, while retaining its private key privC.
Upon receiving pubC, the Server can compute the symmetric key SK using its private key privS from its dynamically generated ECDHE key pair (privS, pubS).
At the same time, by including the attester's identity (raw public key or certificate) in all subsequent messages of the remote attestation process and having the attester (using its attestation evidence signing key) and verifier (using its attestation result signing key) endorse and sign it, a key binding mechanism between the attester's attestation result and its identity is implemented.
After completing the remote attestation with the Verifier, the Server includes its pubS in the Attestation Result returned to the Client.
Once the Client verifies the Attestation Result, it can compute the symmetric key SK using pubS and its own private key privC, thereby completing both the remote attestation and ECDHE key agreement. Additionally, the client can record the binding relationship between the generated SK and the corresponding attester's identity, ensuring that the SK is subsequently used to establish a secure protocol connection with an attester possessing the correct identity.
Some metadata negotiated alongside the key exchange materials between both parties includes: session ID, nonce, algorithm, etc.

## Enterprise Customer Key Distribution to Public Cloud Into Remote Attestation

Enterprises need to deploy data assets on public clouds, such as data, applications, systems, etc., some of which are critical assets of the enterprise, such as private large model applications. It is essential to ensure the trustworthiness and security of the operating environment. The principle is to first upload the encrypted data assets to the cloud environment, then perform remote attestation on the operating environment. Once it passes the security verification, the decryption key is distributed to the operating environment for loading and running the decrypted data assets. In fact, the key here can also be generalized to refer to the decrypted plaintext and the calculated signatures provided by Enterprise KMS. This scenario is similar to the public cloud KMS key distribution scenario, except that the public cloud is no longer responsible for key distribution. Instead, the enterprise manages the keys itself and completes the key distribution after passing the security verification through remote attestation. The method is illustrated in the following diagram:


~~~
            .------------.
            |            | 2. Compare Evidence
            |  Verifier  | against appraisal policy
            |            |
            '--------+---'
                ^    |
  1. Evidence + |    | 3. Attestation
  Attester's    |    | Result + Attester's identity
  identity      |    v
            .---+--------.                .-------------.
            |            +--------------->|Relying Party| 5. Compare Attestation
            |  Attester  |4. Attestation  |& Enterprise | Result against
            |            |   Result +     |      KMS    | appraisal policy
            '------------' Attester's     '-------------' + Distribute Keys
                           identity                         with Attester's public key

~~~
{: #fig-rats-key-negotiation-integration-enterprise-kms title="Enterprise KMS Key Distribution Integrated with Remote Attestation Passport Model Interaction"}

By including the attester's identity (raw public key or certificate) in the messages throughout the remote attestation process and having the attester (using its attestation evidence signing key) and verifier (using its attestation result signing key) endorse and sign it, a key binding mechanism between the attester's attestation result and its identity is implemented. Subsequently, the enterprise KMS can use the identity's public key for key distribution, ensuring that the keys are distributed to the correct attester, thereby eliminating diversion attack. During key rotation, the enterprise KMS can proactively trigger the above process to complete the update and rotation of the new and old keys.

# Use of Negotiated Keys in Different Security Protocols

Through the above process, key negotiation/distribution is completed during the remote attestation process, and is synchronized based on the results of the remote attestation.
If the negotiated key is used for application layer encryption, its specific usage is strongly related to the application and can be very flexible.
When the key is used for the security protocols, such as TLS, IPsec, etc., there can be various ways to integrate and utilize it into security protocol:

* TLS: The negotiated key can be used as a pre-shared key for subsequent TLS handshakes; the key can also be used as an externally imported shared key to participate in TLS Hybrid key exchange {{?I-D.ietf-tls-hybrid-design}}. The integration of all these pre-shared keys into the TLS protocol can be achieved through the hybrid authentication mode of tls_cert_with_extern_psk. This approach not only implements a two-factor authentication using the pre-shared key/sk and the server certificate/attester's identity but also verifies and ensures the correctness of the binding relationship between the sk and the attester's identity. If the server certificate obtained during the TLS negotiation cannot be correctly bound to the currently used pre-shared key, the TLS negotiation will be terminated.
* IPsec: The negotiated key can be used as a pre-shared key for subsequent IKEv2 handshakes; or the key can be directly used as a session key for data plane encryption and integrity protection in IPSec ESP; or the key can also be used as Post-quantum Preshared Keys (PPKs) {{!RFC8784}} to be used in the IPsec protocol. Similar to TLS, the hybrid authentication mode using pre-shared keys and certificates not only achieves two-factor authentication but also ensures the correctness of the binding relationship between the two.

# Remote Attestation Protocol and Message Extensions

This section describes how to extend RATS protocol and message to incorporate key negotiation into the remote attestation process.

TBD

# Security Considerations

TBD

# Privacy Considerations

TBD

# IANA Considerations

TBD

--- back
