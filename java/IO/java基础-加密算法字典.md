# Java Security Standard Algorithm Names

- [Java Security Standard Algorithm Names](#java-security-standard-algorithm-names)
  - [`AlgorithmParameterGenerator` Algorithms](#algorithmparametergenerator-algorithms)
  - [`AlgorithmParameters` Algorithms](#algorithmparameters-algorithms)
  - [`CertificateFactory` Types](#certificatefactory-types)
  - [`CertPath` Encodings](#certpath-encodings)
  - [`CertPathBuilder` Algorithms](#certpathbuilder-algorithms)
  - [`CertPathValidator` Algorithms](#certpathvalidator-algorithms)
  - [`CertStore` Types](#certstore-types)
  - [`Cipher` Algorithms](#cipher-algorithms)
  - [`Cipher` Algorithm Modes](#cipher-algorithm-modes)
  - [`Cipher` Algorithm Paddings](#cipher-algorithm-paddings)
  - [`Configuration` Types](#configuration-types)
  - [Exemption Mechanisms](#exemption-mechanisms)
  - [GSSAPI Mechanisms](#gssapi-mechanisms)
  - [`KDF` Algorithms](#kdf-algorithms)
  - [`KEM` Algorithms](#kem-algorithms)
  - [`Key` Algorithms](#key-algorithms)
    - [`SecretKey` Algorithms](#secretkey-algorithms)
    - [`AsymmetricKey` Algorithms](#asymmetrickey-algorithms)
  - [`Key` Encodings](#key-encodings)
  - [`KeyAgreement` Algorithms](#keyagreement-algorithms)
  - [`KeyFactory` Algorithms](#keyfactory-algorithms)
  - [`KeyGenerator` Algorithms](#keygenerator-algorithms)
  - [`KeyManagerFactory` Algorithms](#keymanagerfactory-algorithms)
  - [`KeyPairGenerator` Algorithms](#keypairgenerator-algorithms)
  - [`KeyStore` Types](#keystore-types)
  - [`Mac` Algorithms](#mac-algorithms)
  - [`MessageDigest` Algorithms](#messagedigest-algorithms)
  - [`ParameterSpec` Names](#parameterspec-names)
    - [`NamedParameterSpec`](#namedparameterspec)
    - [`ECGenParameterSpec`](#ecgenparameterspec)
    - [`PSSParameterSpec`](#pssparameterspec)
  - [`SaslClient` Mechanisms](#saslclient-mechanisms)
  - [`SaslServer` Mechanisms](#saslserver-mechanisms)
  - [`SecretKeyFactory` Algorithms](#secretkeyfactory-algorithms)
  - [`SecureRandom` Number Generation Algorithms](#securerandom-number-generation-algorithms)
  - [Service Attributes](#service-attributes)
  - [`Signature` Algorithms](#signature-algorithms)
  - [`SSLContext` Algorithms](#sslcontext-algorithms)
  - [`TrustManagerFactory` Algorithms](#trustmanagerfactory-algorithms)
  - [XML Signature (`XMLSignatureFactory`/`KeyInfoFactory`/`TransformService`) Mechanisms](#xml-signature-xmlsignaturefactorykeyinfofactorytransformservice-mechanisms)
  - [XML Signature Transform (`TransformService`) Algorithms](#xml-signature-transform-transformservice-algorithms)
  - [JSSE Cipher Suite Names](#jsse-cipher-suite-names)
  - [Additional JSSE Standard Names](#additional-jsse-standard-names)
    - [Key Types](#key-types)
    - [Protocols](#protocols)
    - [Authentication Types](#authentication-types)
    - [Endpoint Identification Algorithms](#endpoint-identification-algorithms)
    - [Signature Schemes](#signature-schemes)
    - [Named Groups](#named-groups)
  - [Security Algorithm Specification](#security-algorithm-specification)
    - [Specification Template](#specification-template)
    - [Algorithm Specifications](#algorithm-specifications)
  - [Security Algorithm Implementation Requirements](#security-algorithm-implementation-requirements)
    - [XML Signature Algorithms](#xml-signature-algorithms)

The Java SE Security API requires and uses a set of standard names
for algorithms, certificate and keystore types. You can find a list of
standard algorithm names in this document.

Note that an SE implementation may support additional algorithms that
are not defined in this specification. As a best practice, if an
algorithm is defined in a subsequent version of this specification and
an implementation of an earlier specification supports that algorithm,
the implementation should use the standard name of the algorithm that is
defined in the subsequent specification. Each SE implementation should
also document the algorithms that it supports or adds support for in
subsequent update releases. The algorithms may be documented in release
notes or in a separate document such as the [JDK
Providers Documentation](https://docs.oracle.com/pls/topic/lookup?ctx=javase25&id=GUID-FE2D2E28-C991-4EF9-9DBE-2A4982726313).

In some cases naming conventions are given for forming names that are
not explicitly listed, to facilitate name consistency across provider
implementations. Items in angle brackets (such as
`<digest>` and `<encryption>`) are
placeholders to be replaced by a specific message digest, encryption
algorithm, or other name.

**Note:** Standard names are not case-sensitive.

**Note:** The [JDK
Providers Documentation](https://docs.oracle.com/pls/topic/lookup?ctx=javase25&id=GUID-FE2D2E28-C991-4EF9-9DBE-2A4982726313) contains specific provider and algorithm
information.

## `AlgorithmParameterGenerator` Algorithms

The algorithm names in this section can be specified when generating
an instance of `AlgorithmParameterGenerator`.

| Algorithm Name | Description |
| --- | --- |
| DiffieHellman | Parameters for use with the Diffie-Hellman algorithm. |
| DSA | Parameters for use with the Digital Signature Algorithm. |

## `AlgorithmParameters` Algorithms

The algorithm names in this section can be specified when generating
an instance of `AlgorithmParameters`.

| Algorithm Name | Description |
| --- | --- |
| AES | Parameters for use with the AES algorithm. |
| Blowfish | Parameters for use with the Blowfish algorithm. |
| ChaCha20-Poly1305 | Parameters for use with the ChaCha20-Poly1305 algorithm, as defined in [RFC 8103](https://tools.ietf.org/html/rfc8103). |
| DES | Parameters for use with the DES algorithm. |
| DESede | Parameters for use with the DESede algorithm. |
| DiffieHellman | Parameters for use with the DiffieHellman algorithm. |
| DSA | Parameters for use with the Digital Signature Algorithm. |
| EC | Parameters for use with the EC algorithm. |
| GCM | Parameters for use with the Galois/Counter Mode (GCM) cipher mode, as defined in [RFC 5084](https://tools.ietf.org/html/rfc5084). |
| OAEP | Parameters for use with the OAEP algorithm. |
| PBEWith<digest>And<encryption>  PBEWith<prf>And<encryption> | Parameters for use with PKCS #5 password-based encryption, where <digest> is a message digest, <prf> is a pseudo-random function, and <encryption> is an encryption algorithm. Examples: **PBEWithMD5AndDES**, and **PBEWithHmacSHA256AndAES**. |
| PBE | Parameters for use with the PBE algorithm. *This name should not be used, in preference to the more specific PBE-algorithm names previously listed.* |
| PBES2 | Parameters for use with the PBES2 password-based encryption algorithm as defined in PKCS #5: Password-Based Cryptography Specification, Version 2.1. |
| RC2 | Parameters for use with the RC2 algorithm. |
| RSASSA-PSS | Parameters for use with the RSASSA-PSS signature algorithm. |

## `CertificateFactory` Types

The type in this section can be specified when generating an instance
of `CertificateFactory`.

| Type | Description |
| --- | --- |
| X.509 | The certificate type defined in X.509, also specified in [RFC 5280](https://tools.ietf.org/html/rfc5280). |

## `CertPath` Encodings

The following encodings may be passed to the `getEncoded`
method of `CertPath` or the
`generateCertPath(InputStream inStream, String encoding)`
method of `CertificateFactory`.

| Encoding | Description |
| --- | --- |
| PKCS7 | A PKCS #7 SignedData object, with the only significant field being certificates. In particular, the signature and the contents are ignored. If no certificates are present, a zero-length `CertPath` is assumed.    **Warning:** PKCS #7 does not maintain the order of certificates in a certification path. This means that if a `CertPath` is converted to PKCS #7 encoded bytes and then converted back, the order of the certificates may change, potentially rendering the `CertPath` invalid. Users should be aware of this behavior.    See [PKCS #7: Cryptographic Message Syntax](https://www.rfc-editor.org/rfc/rfc2315.txt) for details on PKCS #7. |
| PkiPath | An ASN.1 DER encoded sequence of certificates, defined as follows:    `PkiPath ::= SEQUENCE OF Certificate`    Within the sequence, the order of certificates is such that the subject of the first certificate is the issuer of the second certificate, and so on. Each certificate in `PkiPath` shall be unique. No certificate may appear more than once in a value of `Certificate` in `PkiPath`. The `PkiPath` format is defined in defect report 279 against X.509 (2000) and is incorporated into Technical Corrigendum 1 (DTC 2) for the ITU-T Recommendation X.509 (2000). See [the ITU web site](https://www.itu.int/rec/T-REC-X.509/en) for details. |

## `CertPathBuilder` Algorithms

The algorithm in this section can be specified when generating an
instance of `CertPathBuilder`.

| Algorithm Name | Description |
| --- | --- |
| PKIX | The PKIX certification path validation algorithm as defined in the [ValidationAlgorithm service attribute](#service-attributes). The output of `CertPathBuilder` instances implementing this algorithm is a certification path validated against the PKIX validation algorithm. |

## `CertPathValidator` Algorithms

The algorithm in this section can be specified when generating an
instance of `CertPathValidator`.

| Algorithm Name | Description |
| --- | --- |
| PKIX | The PKIX certification path validation algorithm as defined in the [ValidationAlgorithm service attribute](#service-attributes). |

## `CertStore` Types

The types in this section can be specified when generating an
instance of `CertStore`.

| Type | Description |
| --- | --- |
| Collection | A `CertStore` implementation that retrieves certificates and CRLs from a `Collection`. This type of `CertStore` is particularly useful in applications where certificates or CRLs are received in a bag or some sort of attachment, such as with a signed email message or in an SSL negotiation. |
| LDAP | A `CertStore` implementation that fetches certificates and CRLs from an LDAP directory using the schema defined in the [LDAPSchema service attribute](#service-attributes). |

## `Cipher` Algorithms

The following names can be specified as the *algorithm*
component in a [transformation](../../api/java.base/javax/crypto/Cipher.html)
when requesting an instance of `Cipher`.

**Note:** It is recommended to use a transformation that
fully specifies the algorithm, mode, and padding. By not doing so, the
provider will use a default for the mode and padding which may not meet
the security requirements of your application.

| Algorithm Name | Description |
| --- | --- |
| AES | Advanced Encryption Standard as specified by NIST in [FIPS 197](https://csrc.nist.gov/publications/fips/fips197/fips-197.pdf). Also known as the Rijndael algorithm by Joan Daemen and Vincent Rijmen, AES is a 128-bit block cipher supporting keys of 128, 192, and 256 bits.    To use the AES cipher with only one valid key size, use the format AES\_<n>, where <n> can be 128, 192 or 256. |
| AESWrap | The AES key wrapping algorithm as described in [RFC 3394](https://tools.ietf.org/html/rfc3394)  and [NIST Special Publication SP 800-38F](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf).  This is same as specifying AES cipher with KW mode and NoPadding. To use the AESWrap cipher with only one valid key size, use the format AESWrap\_<n>, where <n> can be 128, 192, or 256. |
| AESWrapPad | The AES key wrapping algorithm as described in [RFC 5649](https://tools.ietf.org/html/rfc5649) and  [NIST Special Publication SP 800-38F](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf).  This is same as specifying AES cipher with KWP mode and NoPadding. To use the AESWrapPad cipher with only one valid key size, use the format AESWrapPad\_<n>, where <n> can be 128, 192, or 256. |
| ARCFOUR | A stream cipher believed to be fully interoperable with the RC4 cipher developed by Ron Rivest. For more information, see K. Kaukonen and R. Thayer, ["A Stream Cipher Encryption Algorithm 'Arcfour'"](https://tools.ietf.org/id/draft-kaukonen-cipher-arcfour-03.txt), Internet Draft (expired). |
| Blowfish | The [Blowfish block cipher](https://www.schneier.com/blowfish.html) designed by Bruce Schneier. |
| ChaCha20 | The ChaCha20 stream cipher as defined in [RFC 7539](https://tools.ietf.org/html/rfc7539). |
| ChaCha20-Poly1305 | The ChaCha20 cipher in AEAD mode using the Poly1305 authenticator, as defined in [RFC 7539](https://tools.ietf.org/html/rfc7539). |
| DES | The Digital Encryption Standard as described in [FIPS PUB 46-3](https://csrc.nist.gov/publications/fips/fips46-3/fips46-3.pdf). |
| DESede | Triple DES Encryption (also known as DES-EDE, 3DES, or Triple-DES). Data is encrypted using the DES algorithm three separate times. It is first encrypted using the first subkey, then decrypted with the second subkey, and encrypted with the third subkey. |
| DESedeWrap | The DESede key wrapping algorithm as described in [RFC 3217](https://tools.ietf.org/html/rfc3217). |
| ECIES | Elliptic Curve Integrated Encryption Scheme |
| PBEWith<digest>And<encryption>  PBEWith<prf>And<encryption> | The password-based encryption algorithm defined in PKCS #5, using the specified message digest (<digest>) or pseudo-random function (<prf>) and encryption algorithm (<encryption>). Examples:    **PBEWithMD5AndDES**: The PBES1 password-based encryption algorithm as defined in [PKCS #5: Password-Based Cryptography Specification, Version 2.1](https://tools.ietf.org/html/rfc8018). Note that this algorithm implies [CBC](#cbc) as the cipher mode and [PKCS5Padding](#pkcs5padding) as the padding scheme and cannot be used with any other cipher modes or padding schemes.    **PBEWithHmacSHA256AndAES\_128**: The PBES2 password-based encryption algorithm as defined in [PKCS #5: Password-Based Cryptography Specification, Version 2.1](https://tools.ietf.org/html/rfc8018). |
| RC2 | Variable-key-size encryption algorithms developed by Ron Rivest for RSA Data Security, Inc. |
| RC4 | Variable-key-size encryption algorithms developed by Ron Rivest for RSA Data Security, Inc. (See note prior for ARCFOUR.) |
| RC5 | Variable-key-size encryption algorithms developed by Ron Rivest for RSA Data Security, Inc. |
| RSA | The RSA encryption algorithm as defined in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). |

## `Cipher` Algorithm Modes

The following names can be specified as the *mode* component
in a [transformation](../../api/java.base/javax/crypto/Cipher.html)
when requesting an instance of `Cipher`.

| Algorithm Name | Description |
| --- | --- |
| NONE | No mode. |
| CBC | Cipher Block Chaining Mode, as defined in [FIPS PUB 81](https://csrc.nist.gov/publications/fips/fips81/fips81.htm). |
| CCM | Counter/CBC Mode, as defined in [NIST Special Publication SP 800-38C](https://csrc.nist.gov/publications/nistpubs/800-38C/SP800-38C_updated-July20_2007.pdf). |
| CFB, CFBx | Cipher Feedback Mode, as defined in [FIPS PUB 81](https://csrc.nist.gov/publications/fips/fips81/fips81.htm).    Using modes such as CFB and OFB, block ciphers can encrypt data in units smaller than the cipher's actual block size. When requesting such a mode, you may optionally specify the number of bits to be processed at a time by appending this number to the mode name as shown in the "*DES/CFB8/NoPadding*" and "*DES/OFB32/PKCS5Padding*" transformations. If no such number is specified, a provider-specific default is used. (For example, the SunJCE provider uses a default of 64 bits for DES.) Thus, block ciphers can be turned into byte-oriented stream ciphers by using an 8-bit mode such as CFB8 or OFB8. |
| CTR | A simplification of OFB, Counter mode updates the input block as a counter. |
| CTS | Cipher Text Stealing, as described in Bruce Schneier's book *Applied Cryptography-Second Edition*, John Wiley and Sons, 1996. |
| ECB | Electronic Codebook Mode, as defined in [FIPS PUB 81](https://csrc.nist.gov/publications/fips/fips81/fips81.htm) (generally this mode should not be used for multiple blocks of data). |
| GCM | Galois/Counter Mode, as defined in [NIST Special Publication SP 800-38D](https://csrc.nist.gov/publications/nistpubs/800-38D/SP-800-38D.pdf). |
| KW | Key Wrap (KW) mode, as defined in [RFC 3394](https://tools.ietf.org/html/rfc3394) and [NIST Special Publication SP 800-38F](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf). |
| KWP | Key Wrap With Padding (KWP) mode, as defined in [RFC 5649](https://tools.ietf.org/html/rfc5649) and [NIST Special Publication SP 800-38F](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38F.pdf). |
| OFB, OFBx | Output Feedback Mode, as defined in [FIPS PUB 81](https://csrc.nist.gov/publications/fips/fips81/fips81.htm).    Using modes such as CFB and OFB, block ciphers can encrypt data in units smaller than the cipher's actual block size. When requesting such a mode, you may optionally specify the number of bits to be processed at a time by appending this number to the mode name as shown in the "*DES/CFB8/NoPadding*" and "*DES/OFB32/PKCS5Padding*" transformations. If no such number is specified, a provider-specific default is used. (For example, the SunJCE provider uses a default of 64 bits for DES.) Thus, block ciphers can be turned into byte-oriented stream ciphers by using an 8-bit mode such as CFB8 or OFB8. |
| PCBC | Propagating Cipher Block Chaining, as defined by [Kerberos V4](https://web.mit.edu/kerberos/). |

## `Cipher` Algorithm Paddings

The following names can be specified as the *padding*
component in a [transformation](../../api/java.base/javax/crypto/Cipher.html)
when requesting an instance of `Cipher`.

| Algorithm Name | Description |
| --- | --- |
| NoPadding | No padding. |
| ISO10126Padding | This padding for block ciphers is described in the [ISO 10126 standard](https://www.iso.org/standard/18114.html) (now withdrawn). |
| OAEPPadding, OAEPWith<digest>And<mgf>Padding | Optimal Asymmetric Encryption. Padding scheme defined in PKCS #1, where <digest> should be replaced by the message digest and <mgf> by the mask generation function. Examples: **OAEPWithMD5AndMGF1Padding** and **OAEPWithSHA-512AndMGF1Padding**.    If `OAEPPadding` is used, `Cipher` objects are initialized with a `javax.crypto.spec.OAEPParameterSpec` object to supply values needed for OAEPPadding. |
| PKCS1Padding | The padding scheme described in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017), used with the RSA algorithm. |
| PKCS5Padding | The padding scheme described in [PKCS #5: Password-Based Cryptography Specification, version 2.1](https://tools.ietf.org/html/rfc8018). |
| SSL3Padding | The padding scheme defined in the SSL Protocol Version 3.0, November 18, 1996, section 5.2.3.2 (CBC block cipher):    `block-ciphered struct {`      `opaque content[SSLCompressed.length];`      `opaque MAC[CipherSpec.hash_size];`      `uint8 padding[GenericBlockCipher.padding_length];`      `uint8 padding_length;`  `} GenericBlockCipher;`    The size of an instance of a `GenericBlockCipher` must be a multiple of the block cipher's block length. The padding length, which is always present, contributes to the padding, which implies that if:    `sizeof(content) + sizeof(MAC) % block_length = 0,`    padding has to be `(block_length - 1)` bytes long, because of the existence of `padding_length`.    This makes the padding scheme similar (but not quite) to PKCS5Padding, where the padding length is encoded in the padding (and ranges from 1 to `block_length`). With the SSL scheme, the `sizeof(padding)` is encoded in the always present `padding_length` and therefore ranges from 0 to `block_length-1`. |

## `Configuration` Types

The type in this section can be specified when generating an instance
of `javax.security.auth.login.Configuration`.

| Type | Description |
| --- | --- |
| JavaLoginConfig | The default Configuration implementation from the SUN provider, as described in the [Configuration class specification](../../api/java.base/javax/security/auth/login/Configuration.html). This type accepts `java.security.URIParameter` as a valid `Configuration.Parameter` type. If this parameter is not specified, then the configuration information is loaded from the sources described in the `ConfigFile` class specification. If this parameter is specified, the configuration information is loaded solely from the specified URI. |

## Exemption Mechanisms

The following exemption mechanism names can be specified in the
permission policy file that accompanies an application considered
"exempt" from cryptographic restrictions.

| Algorithm Name | Description |
| --- | --- |
| KeyEscrow | An encryption system with a backup decryption capability that allows authorized persons (users, officers of an organization, and government officials), under certain prescribed conditions, to decrypt ciphertext with the help of information supplied by one or more trusted parties who hold special data recovery keys. |
| KeyRecovery | A method of obtaining the secret key used to lock encrypted data. One use is as a means of providing fail-safe access to a corporation's own encrypted information in times of disaster. |
| KeyWeakening | A method in which a part of the key can be escrowed or recovered. |

## GSSAPI Mechanisms

The following mechanisms can be specified when using GSSAPI. Note
that Object Identifiers (OIDs) are specified instead of names to be
consistent with the GSSAPI standard.

| Mechanism OID | Description |
| --- | --- |
| 1.2.840.113554.1.2.2 | The Kerberos v5 GSS-API mechanism defined in [RFC 4121](https://tools.ietf.org/html/rfc4121). |
| 1.3.6.1.5.5.2 | The Simple and Protected GSS-API Negotiation (SPNEGO) mechanism defined in [RFC 4178](https://tools.ietf.org/html/rfc4178). |

## `KDF` Algorithms

The algorithm names in this section can be specified when requesting
an instance of `KDF`.

| Algorithm Name | Description |
| --- | --- |
| HKDF-SHA256  HKDF-SHA384  HKDF-SHA512 | HMAC-based KDF as defined in [RFC 5869](https://www.rfc-editor.org/rfc/rfc5869). |

## `KEM` Algorithms

The algorithm names in this section can be specified when generating
an instance of `KEM`.

| Algorithm Name | Description |
| --- | --- |
| DHKEM | DH-Based KEM as defined in [RFC 9180](https://www.rfc-editor.org/rfc/rfc9180#name-dh-based-kem-dhkem). |
| ML-KEM | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). This algorithm supports keys with ML-KEM-512, ML-KEM-768, and ML-KEM-1024 parameter sets. |
| ML-KEM-512 | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-512 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-768 | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-768 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-1024 | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-1024 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |

## `Key` Algorithms

The algorithm names returned by `Key.getAlgorithm()`.

**Note:** The algorithm names of keys generated or
created by the `KeyFactory`, `SecretKeyFactory`,
`KeyGenerator`, or `KeyPairGenerator` APIs should
use the standard names listed below, even if an alias or a standard
parameter set name was used to instantiate the service. For example,
`KeyPairGenerator.getInstance("DiffieHellman")` should
generate a key with the "DH" algorithm name, and
`KeyPairGenerator.getInstance("ML-KEM-768")` should generate
a key with the "ML-KEM" algorithm name. However, some third-party
providers may use the alias or a non-standard name as the algorithm
name.

### `SecretKey` Algorithms

| Algorithm Name | Description |
| --- | --- |
| AES | The Advanced Encryption Standard (AES) algorithm. |
| ARCFOUR | The ARCFOUR (RC4) algorithm. |
| Blowfish | The Blowfish algorithm. |
| ChaCha20 | The ChaCha20 algorithm. |
| DES | The Data Encryption Standard (DES) algorithm. |
| DESede | The Triple Data Encryption Algorithm (DESede or Triple-DES). |
| Generic | This algorithm defines a general-purpose secret key. This key is not intended to be used for cryptographic operations; rather it is typically used as input keying material for deriving other keys. This algorithm is also used to represent PKCS #11 generic secret key objects (key type CKK\_GENERIC\_SECRET\_TYPE). |
| PBEWith<digest>And<encryption>  PBEWith<prf>And<encryption> | PKCS #5 password-based encryption, where <digest> is a message digest, <prf> is a pseudo-random function, and <encryption> is an encryption algorithm. Examples: **PBEWithMD5AndDES**, and **PBEWithHmacSHA256AndAES**. |
| PBKDF2With<prf> | Password-based key-derivation algorithm defined in [PKCS #5: Password-Based Cryptography Specification, Version 2.1](https://tools.ietf.org/html/rfc8018) using the specified pseudo-random function (<prf>). Example: **PBKDF2WithHmacSHA256**. |
| RC2 | The RC2 algorithm. |

### `AsymmetricKey` Algorithms

| Algorithm Name | Description |
| --- | --- |
| DH | The Diffie-Hellman KeyAgreement algorithm. |
| DSA | The (original) Digital Signature Algorithm. |
| EC | The Elliptic Curve algorithm. |
| EdDSA | The Edwards-Curve signature algorithm with elliptic curves as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| HSS/LMS | The Leighton-Micali Signature (LMS) system with the Hierarchical Signature System (HSS) as defined in [RFC 8554](https://tools.ietf.org/html/rfc8554). |
| ML-DSA | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-KEM | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| RSA | The Rivest-Shamir-Adleman (RSA) algorithm. |
| RSASSA-PSS | The RSA Signature Scheme with Appendix - Probabilistic Signature Scheme (RSASSA-PSS) signature algorithm. |
| XDH | The Diffie-Hellman key agreement with elliptic curves as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |

## `Key` Encodings

The names of primary encoding formats returned by [Key.getFormat()](../../api/java.base/java/security/Key.html#getFormat())
or [EncodedKeySpec.getFormat()](../../api/java.base/java/security/spec/EncodedKeySpec.html#getFormat()).

| Encoding | Description |
| --- | --- |
| PKCS#1 | The ASN.1 data format for `RSAPrivateKey` as defined in [PKCS #1](https://www.rfc-editor.org/rfc/rfc8017.html). |
| PKCS#8 | The ASN.1 data format for `PrivateKeyInfo`, as defined in [PKCS #8](https://www.rfc-editor.org/rfc/rfc5208.html). |
| RAW | The raw key bytes. |
| X.509 | The ASN.1 data format for `SubjectPublicKeyInfo`, as defined by X.509, and also specified in [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280.html). |

## `KeyAgreement` Algorithms

The following algorithm names can be specified when requesting an
instance of `KeyAgreement`.

| Algorithm Name | Description |
| --- | --- |
| DiffieHellman | Diffie-Hellman Key Agreement as defined in PKCS #3: Diffie-Hellman Key-Agreement Standard, RSA Laboratories, version 1.4, November 1993. |
| ECDH | Elliptic Curve Diffie-Hellman as defined in ANSI X9.63. |
| ECMQV | Elliptic Curve Menezes-Qu-Vanstone. |
| XDH | Diffie-Hellman key agreement with elliptic curves as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |
| X25519 | Diffie-Hellman key agreement with Curve25519 as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |
| X448 | Diffie-Hellman key agreement with Curve448 as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |

## `KeyFactory` Algorithms

The algorithm names in this section can be specified when generating
an instance of `KeyFactory`.

*(Except as noted, these classes create keys for which [Key.getAlgorithm()](../../api/java.base/java/security/Key.html#getAlgorithm())
returns the standard algorithm name.)*

| Algorithm Name | Description |
| --- | --- |
| DiffieHellman | Keys for the Diffie-Hellman KeyAgreement algorithm.    **Note:** `key.getAlgorithm()` will return "DH" instead of "DiffieHellman". |
| DSA | Keys for the Digital Signature Algorithm. |
| EC | Keys for the Elliptic Curve algorithm. |
| EdDSA | Keys for Edwards-Curve signature algorithm with elliptic curves as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| Ed25519 | Keys for Edwards-Curve signature algorithm with Ed25519 as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| Ed448 | Keys for Edwards-Curve signature algorithm with Ed448 as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| HSS/LMS | Keys for the Leighton-Micali Signature (LMS) system with the Hierarchical Signature System (HSS) as defined in [RFC 8554](https://tools.ietf.org/html/rfc8554). |
| ML-DSA | Keys for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). This algorithm supports keys with ML-DSA-44, ML-DSA-65, and ML-DSA-87 parameter sets. |
| ML-DSA-44 | Keys for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-44 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-65 | Keys for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-65 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-87 | Keys for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-87 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-KEM | Keys for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). This algorithm supports keys with ML-KEM-512, ML-KEM-768, and ML-KEM-1024 parameter sets. |
| ML-KEM-512 | Keys for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-512 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-768 | Keys for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-768 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-1024 | Keys for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-1024 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| RSA | Keys for the RSA algorithm (Signature/Cipher). |
| RSASSA-PSS | Keys for the RSASSA-PSS algorithm (Signature). |
| XDH | Keys for Diffie-Hellman key agreement with elliptic curves as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |
| X25519 | Keys for Diffie-Hellman key agreement with Curve25519 as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |
| X448 | Keys for Diffie-Hellman key agreement with Curve448 as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |

## `KeyGenerator` Algorithms

The following algorithm names can be specified when requesting an
instance of `KeyGenerator`.

*(These classes generate keys for which [Key.getAlgorithm()](../../api/java.base/java/security/Key.html#getAlgorithm())
returns the standard algorithm name.)*

| Algorithm Name | Description |
| --- | --- |
| AES | Key generator for use with the AES algorithm. |
| ARCFOUR | Key generator for use with the ARCFOUR (RC4) algorithm. |
| Blowfish | Key generator for use with the Blowfish algorithm. |
| ChaCha20 | Key generator for use with the ChaCha20 and ChaCha20-Poly1305 algorithms. |
| DES | Key generator for use with the DES algorithm. |
| DESede | Key generator for use with the DESede (triple-DES) algorithm. |
| HmacMD5 | Key generator for use with the HmacMD5 algorithm. |
| HmacSHA1  HmacSHA224  HmacSHA256  HmacSHA384  HmacSHA512  HmacSHA512/224  HmacSHA512/256  HmacSHA3-224  HmacSHA3-256  HmacSHA3-384  HmacSHA3-512 | Key generator for use with the various flavors of the HmacSHA algorithms. |
| RC2 | Key generator for use with the RC2 algorithm. |

## `KeyManagerFactory` Algorithms

The algorithm names that can be specified when generating an instance
of `KeyManagerFactory`.

| Algorithm Name | Description |
| --- | --- |
| PKIX | A factory for `X509ExtendedKeyManager`s that manage X.509 certificate-based key pairs for local side authentication according to the rules defined by the IETF PKIX working group in [RFC 5280](https://tools.ietf.org/html/rfc5280) or its successor. The `KeyManagerFactory` must support initialization using the class `javax.net.ssl.KeyStoreBuilderParameters`. |

## `KeyPairGenerator` Algorithms

The algorithm names that can be specified when generating an instance
of `KeyPairGenerator`.

*(Except as noted, these classes create keys for which [Key.getAlgorithm()](../../api/java.base/java/security/Key.html#getAlgorithm())
returns the standard algorithm name.)*

| Algorithm Name | Description |
| --- | --- |
| DiffieHellman | Generates keypairs for the Diffie-Hellman KeyAgreement algorithm.    **Note:** `key.getAlgorithm()` will return "DH" instead of "DiffieHellman". |
| DSA | Generates keypairs for the Digital Signature Algorithm. |
| RSA | Generates keypairs for the RSA algorithm (Signature/Cipher). |
| RSASSA-PSS | Generates keypairs for the RSASSA-PSS signature algorithm. |
| EC | Generates keypairs for the Elliptic Curve algorithm. |
| EdDSA | Generates keypairs for Edwards-Curve signature algorithm with elliptic curves as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| Ed25519 | Generates keypairs for Edwards-Curve signature algorithm with Ed25519 as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| Ed448 | Generates keypairs for Edwards-Curve signature algorithm with Ed448 as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| ML-DSA | Generates keypairs for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). This algorithm supports keys with ML-DSA-44, ML-DSA-65, and ML-DSA-87 parameter sets. |
| ML-DSA-44 | Generates keypairs for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-44 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-65 | Generates keypairs for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-65 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-87 | Generates keypairs for the Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-87 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-KEM | Generates keypairs for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). This algorithm supports keys with ML-KEM-512, ML-KEM-768, and ML-KEM-1024 parameter sets. |
| ML-KEM-512 | Generates keypairs for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-512 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-768 | Generates keypairs for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-768 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-1024 | Generates keypairs for the Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-1024 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| XDH | Generates keypairs for Diffie-Hellman key agreement with elliptic curves as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |
| X25519 | Generates keypairs for Diffie-Hellman key agreement with Curve25519 as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |
| X448 | Generates keypairs for Diffie-Hellman key agreement with Curve448 as defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |

**Note:** For `KeypairGenerator` algorithms
that use `NamedParameterSpec` names, the
`key.getAlgorithm()` for generated keys will return the
"family" algorithm name instead of the specific parameter set name. For
example, `KeyPairGenerator.getInstance("ML-KEM-768")` will
generate a key with the algorithm name "ML-KEM".

## `KeyStore` Types

The types in this section can be specified when generating an
instance of `KeyStore`.

| Type | Description |
| --- | --- |
| jceks | The proprietary keystore implementation provided by the SunJCE provider. |
| jks | The proprietary keystore implementation provided by the SUN provider. |
| dks | A domain keystore is a collection of keystores presented as a single logical keystore. It is specified by configuration data whose syntax is described in the [DomainLoadStoreParameter](../../api/java.base/java/security/DomainLoadStoreParameter.html) class. |
| pkcs11 | A keystore backed by a PKCS #11 token. |
| pkcs12 | The transfer syntax for personal identity information as defined in [PKCS #12](https://tools.ietf.org/html/rfc7292). |

## `Mac` Algorithms

The following algorithm names can be specified when requesting an
instance of `Mac`.

| Algorithm Name | Description |
| --- | --- |
| HmacMD5 | The HMAC-MD5 keyed-hashing algorithm as defined in [RFC 2104](https://tools.ietf.org/html/rfc2104): "HMAC: Keyed-Hashing for Message Authentication" (February 1997). |
| HmacSHA1  HmacSHA224  HmacSHA256  HmacSHA384  HmacSHA512  HmacSHA512/224  HmacSHA512/256  HmacSHA3-224  HmacSHA3-256  HmacSHA3-384  HmacSHA3-512 | The HmacSHA\* algorithms as defined in [RFC 2104](https://tools.ietf.org/html/rfc2104) "HMAC: Keyed-Hashing for Message Authentication" (February 1997) with `SHA-*` with SHA, SHA-2, and SHA-3 family of digest algorithms. |
| PBEWith<mac> | The PBMAC1 password-based message authentication scheme as defined in [PKCS #5: Password-Based Cryptography Specification, Version 2.1](https://tools.ietf.org/html/rfc8018), where <mac> is a Message Authentication Code algorithm name. Example: **PBEWithHmacSHA256** |
| HmacPBESHA1  HmacPBESHA224  HmacPBESHA256  HmacPBESHA384  HmacPBESHA512  HmacPBESHA512/224  HmacPBESHA512/256 | The HMAC algorithms as defined in [Appendix B.4 of RFC 7292](https://tools.ietf.org/html/rfc7292#appendix-B.4): "PKCS #12: Personal Information Exchange Syntax v1.1" (July 2014). |

## `MessageDigest` Algorithms

Algorithm names that can be specified when generating an instance of
`MessageDigest`.

| Algorithm Name | Description |
| --- | --- |
| MD2 | The MD2 message digest algorithm as defined in [RFC 1319](https://tools.ietf.org/html/rfc1319). |
| MD5 | The MD5 message digest algorithm as defined in [RFC 1321](https://tools.ietf.org/html/rfc1321). |
| SHA-1  SHA-224  SHA-256  SHA-384  SHA-512  SHA-512/224  SHA-512/256 | Secure hash algorithms as defined in [FIPS PUB 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf).    SHA-1 produces a 160 bit digest.  SHA-224 produces a 224 bit digest.  SHA-256 produces a 256 bit digest.  SHA-384 produces a 384 bit digest.  SHA-512 produces a 512 bit digest.  SHA-512/224 produces a 224 bit digest.  SHA-512/256 produces a 256 bit digest. |
| SHA3-224  SHA3-256  SHA3-384  SHA3-512  SHAKE128-256  SHAKE256-512 | Permutation-based hash and extendable-output functions as defined in [FIPS PUB 202](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf). An input message length can vary; the length of the output digest is fixed.    SHA3-224 produces a 224 bit digest.  SHA3-256 produces a 256 bit digest.  SHA3-384 produces a 384 bit digest.  SHA3-512 produces a 512 bit digest.  SHAKE128-256 produces a 256 bit digest.  SHAKE256-512 produces a 512 bit digest. |

## `ParameterSpec` Names

### `NamedParameterSpec`

The `NamedParameterSpec` class in the
`java.security.spec` package may be used to specify a set of
parameters using the following names.

| Name | Description |
| --- | --- |
| Ed25519 | Elliptic curve signature scheme using the edwards25519 curve defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| Ed448 | Elliptic curve signature scheme using the edwards448 curve defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| ML-DSA-44 | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-44 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-65 | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-65 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-87 | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-87 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-KEM-512 | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-512 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-768 | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-768 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| ML-KEM-1024 | The Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) using the ML-KEM-1024 parameter set as defined in [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). |
| X25519 | Elliptic curve cryptography using the X25519 scalar multiplication function defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |
| X448 | Elliptic curve cryptography using the X448 scalar multiplication function defined in [RFC 7748](https://tools.ietf.org/html/rfc7748). |

### `ECGenParameterSpec`

The `ECGenParameterSpec` class in the
`java.security.spec` package may be used to specify a set of
elliptic curve parameters using the following names.

| Name | Description |
| --- | --- |
| sect163k1  sect163r1  sect163r2  sect193r1  sect193r2  sect233k1  sect233r1  sect239k1  sect283k1  sect283r1  sect409k1  sect409r1  sect571k1  sect571r1  secp160k1  secp160r1  secp160r2  secp192k1  secp192r1  secp224k1  secp224r1  secp256k1  secp256r1  secp384r1  secp521r1 | The named curves as specified in [SECG, SEC 2: Recommended Elliptic Curve Domain Parameters](https://www.secg.org/sec2-v2.pdf). |
| brainpoolP256r1  brainpoolP384r1  brainpoolP512r1 | The named curves as defined in [RFC 5639](https://tools.ietf.org/html/rfc5639). |

### `PSSParameterSpec`

The `PSSParameterSpec` class in the
`java.security.spec` package may be used to specify the hash
and mask generation function algorithms for the RSASSA-PSS signature
algorithm using the following names.

| Hash Algorithm Name | Description |
| --- | --- |
| SHA-1  SHA-224  SHA-256  SHA-384  SHA-512  SHA-512/224  SHA-512/256 | The hash algorithms as specified in [Appendix A.2.3 of RFC 8017](https://datatracker.ietf.org/doc/html/rfc8017#appendix-A.2.3). |

| MGF Algorithm Name | Description |
| --- | --- |
| MGF1 | The mask generation functions as specified in [Appendix A.2.3 of RFC 8017](https://datatracker.ietf.org/doc/html/rfc8017#appendix-A.2.3). |

## `SaslClient` Mechanisms

The mechanisms in this section can be specified when generating an
instance of `SaslClient`.

| Mechanism | Description |
| --- | --- |
| CRAM-MD5 | See [RFC 2195](https://tools.ietf.org/html/rfc2195). This mechanism supports a hashed user name/password authentication scheme. |
| DIGEST-MD5 | See [RFC 2831](https://tools.ietf.org/html/rfc2831). This mechanism defines how HTTP Digest Authentication can be used as a SASL mechanism. |
| EXTERNAL | See [RFC 2222](https://tools.ietf.org/html/rfc2222). This mechanism obtains authentication information from an external channel (such as TLS or IPsec). |
| GSSAPI | See [RFC 2222](https://tools.ietf.org/html/rfc2222). This mechanism uses the GSSAPI for obtaining authentication information. It supports Kerberos v5 authentication. |
| NTLM | See [MS-NLMP](https://msdn.microsoft.com/en-us/library/cc236621.aspx). This mechanism supports the NTLM authentication scheme. |
| PLAIN | See [RFC 2595](https://tools.ietf.org/html/rfc2595). This mechanism supports cleartext user name/password authentication. |

## `SaslServer` Mechanisms

The mechanisms in this section can be specified when generating an
instance of `SaslServer`.

| Mechanism | Description |
| --- | --- |
| CRAM-MD5 | See [RFC 2195](https://tools.ietf.org/html/rfc2195). This mechanism supports a hashed user name/password authentication scheme. |
| DIGEST-MD5 | See [RFC 2831](https://tools.ietf.org/html/rfc2831). This mechanism defines how HTTP Digest Authentication can be used as a SASL mechanism. |
| GSSAPI | See [RFC 2222](https://tools.ietf.org/html/rfc2222). This mechanism uses the GSSAPI for obtaining authentication information. It supports Kerberos v5 authentication. |
| NTLM | See [MS-NLMP](https://msdn.microsoft.com/en-us/library/cc236621.aspx). This mechanism supports the NTLM authentication scheme. |

## `SecretKeyFactory` Algorithms

The following algorithm names can be specified when requesting an
instance of `SecretKeyFactory`.

*(These classes create keys for which [Key.getAlgorithm()](../../api/java.base/java/security/Key.html#getAlgorithm())
returns the standard algorithm name.)*

| Algorithm Name | Description |
| --- | --- |
| AES | Constructs secret keys for use with the AES algorithm. |
| ARCFOUR | Constructs secret keys for use with the ARCFOUR algorithm. |
| ChaCha20 | Constructs secret keys for use with the ChaCha20 and ChaCha20-Poly1305 algorithms. |
| DES | Constructs secret keys for use with the DES algorithm. |
| DESede | Constructs secret keys for use with the DESede (Triple-DES) algorithm. |
| Generic | Constructs general-purpose secret keys. This `SecretKeyFactory` is primarily useful when working with hardware-based security modules or PKCS #11 tokens where keys are non-extractable. For PKCS#11, these are CKK\_GENERIC\_SECRET keys used for purposes such as Mac algorithms, HKDF Initial Key Material (IKM), HKDF Salt, and other cryptographic operations in a PKCS #11 token.  **Important:** Using this factory in software-based environments can lead to unexpected behavior. If the provider is PKCS #11, the key created may not be extractable and could fail when used with software-based cryptographic operations. For most software-based security providers, creating a `SecretKeySpec` object is a safer and more convenient option. |
| PBEWith<digest>And<encryption>  PBEWith<prf>And<encryption> | Secret-key factory for use with PKCS #5 password-based encryption, where <digest> is a message digest, <prf> is a pseudo-random function, and <encryption> is an encryption algorithm. Examples:    **PBEWithMD5AndDES** (PKCS #5, PBES1 encryption scheme),  **PBEWithHmacSHA256AndAES\_128** (PKCS #5, PBES2 encryption scheme)    **Note:** These all use only the low order 8 bits of each password character. |
| PBKDF2With<prf> | Password-based key-derivation algorithm defined in [PKCS #5: Password-Based Cryptography Specification, Version 2.1](https://tools.ietf.org/html/rfc8018) using the specified pseudo-random function (<prf>). Example:  **PBKDF2WithHmacSHA256**. |

## `SecureRandom` Number Generation Algorithms

The algorithm names in this section can be specified when generating
an instance of `SecureRandom`.

| Algorithm Name | Description |
| --- | --- |
| NativePRNG | Obtains random numbers from the underlying native OS. No assertions are made as to the blocking nature of generating these numbers. |
| NativePRNGBlocking | Obtains random numbers from the underlying native OS, blocking if necessary. For example, `/dev/random` on UNIX-like systems. |
| NativePRNGNonBlocking | Obtains random numbers from the underlying native OS, without blocking to prevent applications from excessive stalling. For example, `/dev/urandom` on UNIX-like systems. |
| PKCS11 | Obtains random numbers from the underlying installed and configured PKCS #11 library. |
| DRBG | An algorithm using DRBG mechanisms as defined in [NIST SP 800-90Ar1](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-90Ar1.pdf). |
| SHA1PRNG | The name of the pseudo-random number generation (PRNG) algorithm supplied by the SUN provider. This algorithm uses SHA-1 as the foundation of the PRNG. It computes the SHA-1 hash over a true-random seed value concatenated with a 64-bit counter which is incremented by 1 for each operation. From the 160-bit SHA-1 output, only 64 bits are used. |
| Windows-PRNG | Obtains random numbers from the underlying Windows OS. |

## Service Attributes

The attributes in this section are for cryptographic services. The
service attributes can be used as filters for selecting providers.

A cryptographic service is always associated with a particular
algorithm or type. For example, a digital signature service is always
associated with a particular algorithm (for example, DSA), and a
`CertificateFactory` service is always associated with a
particular certificate type (for example, X.509).

**Note:** The attribute name and value are
case-insensitive.

| Attribute | Description |
| --- | --- |
| KeySize | The maximum key size that the provider supports for the cryptographic service. |
| ImplementedIn | Whether the implementation for the cryptographic service is done by software or hardware. The value of this attribute is "software" or "hardware". |
| LDAPSchema | The name of the specification that defines the LDAP schema that an implementation of an LDAP `CertStore` uses to retrieve certificates and CRLs. RFCs should be specified as "RFC#" (ex: "RFC2587") and Internet Drafts as the name of the draft. All LDAP implementations of `CertStore` should provide a value for this attribute. |
| SupportedKeyClasses | The list of key classes supported by the cryptographic service. The value is a list of fully qualified class names separated by vertical bars ("|"). For example, when a `Cipher` service provides this attribute, the value indicates the kinds of keys that callers should use. |
| SupportedKeyFormats | The list of key formats supported by the cryptographic service. The value is a list of key format names separated by vertical bars ("|"). Possible key format names include those listed in the [Key Encodings](#key-encodings) section. |
| SupportedModes | The list of modes supported by the cryptographic service. The value is a list of cipher algorithm mode names separated by vertical bars ("|"). Possible mode names include those listed in the [Cipher Algorithm Modes] (#cipher-algorithm-modes) section. |
| SupportedPaddings | The list of paddings supported by the cryptographic service. The value is a list of padding names separated by vertical bars ("|"). Possible padding names include those listed in the [Cipher Algorithm Paddings] (#cipher-algorithm-paddings) section. |
| ThreadSafe | Whether a `SecureRandom` implementation has its `SecureRandomSpi` engine methods implemented thread safe. The value of this attribute is "true" or "false". |
| ValidationAlgorithm | The name of the specification that defines the certification path validation algorithm that an implementation of `CertPathBuilder` or `CertPathValidator` supports. RFCs should be specified as "RFC#" (ex: "RFC5280") and Internet Drafts as the name of the draft (ex: "`draft-ietf-pkix-rfc2560bis-01.txt`"). Values for this attribute that are specified as selection criteria to the `Security.getProviders` method will be compared using the `String.equalsIgnoreCase` method. All PKIX implementations of `CertPathBuilder` and `CertPathValidator` should provide a value for this attribute. |

For example,

```
    map.put("KeyPairGenerator.DSA",
            "sun.security.provider.DSAKeyPairGenerator");
    map.put("KeyPairGenerator.DSA KeySize", "2048");
    map.put("KeyPairGenerator.DSA ImplementedIn", "Software");
```

## `Signature` Algorithms

The algorithm names in this section can be specified when generating
an instance of `Signature`.

| Algorithm Name | Description |
| --- | --- |
| EdDSA | Edwards-Curve signature algorithm as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| Ed25519 | Edwards-Curve signature algorithm with Ed25519 as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| Ed448 | Edwards-Curve signature algorithm with Ed448 as defined in [RFC 8032](https://tools.ietf.org/html/rfc8032). |
| HSS/LMS | The Leighton-Micali Signature (LMS) system with the Hierarchical Signature System (HSS) as defined in [RFC 8554](https://tools.ietf.org/html/rfc8554). |
| ML-DSA | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). This algorithm supports keys with ML-DSA-44, ML-DSA-65, and ML-DSA-87 parameter sets. |
| ML-DSA-44 | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-44 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-65 | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-65 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| ML-DSA-87 | The Module-Lattice-Based Digital Signature Algorithm (ML-DSA) using the ML-DSA-87 parameter set as defined in [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). |
| NONEwithRSA | The RSA signature algorithm which does not use any digesting algorithm and uses only the RSASP1/RSAVP1 primitives as defined in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). |
| MD2withRSA  MD5withRSA | The RSA signature algorithm that uses the MD2/MD5 digest with the RSASSA-PKCS1-v1\_5 signature scheme as defined in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). |
| SHA1withRSA  SHA224withRSA  SHA256withRSA  SHA384withRSA  SHA512withRSA  SHA512/224withRSA  SHA512/256withRSA  SHA3-224withRSA  SHA3-256withRSA  SHA3-384withRSA  SHA3-512withRSA | The RSA signature algorithm that uses the SHA-\* digest with the RSASSA-PKCS1-v1\_5 signature scheme as defined in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). |
| RSASSA-PSS | The signature algorithm that uses the RSASSA-PSS signature scheme as defined in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). Note that this signature algorithm needs parameters such as a hash algorithm, salt length and MGF algorithm, to be supplied with a [`PSSParameterSpec`](../../api/java.base/java/security/spec/PSSParameterSpec.html) object before performing the RSA operation. See the [PSSParameterSpec](#pssparameterspec) section for the standard algorithm names that can be specified. |
| NONEwithDSA | The Digital Signature Algorithm as defined in [FIPS PUB 186-2](https://csrc.nist.gov/publications/fips/archive/fips186-2/fips186-2.pdf). The data must be exactly 20 bytes in length. This algorithm is also known as rawDSA. |
| SHA1withDSA  SHA224withDSA  SHA256withDSA  SHA384withDSA  SHA512withDSA  SHA3-224withDSA  SHA3-256withDSA  SHA3-384withDSA  SHA3-512withDSA | The DSA signature algorithms that use the SHA-1, SHA-2, and SHA-3 family of digest algorithms to create and verify digital signatures as defined in [FIPS PUB 186-3](https://csrc.nist.gov/publications/fips/fips186-3/fips_186-3.pdf) and [FIPS PUB 186-4](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-4.pdf). |
| NONEwithECDSA  SHA1withECDSA  SHA224withECDSA  SHA256withECDSA  SHA384withECDSA  SHA512withECDSA  *(ECDSA)*  SHA3-224withECDSA  SHA3-256withECDSA  SHA3-384withECDSA  SHA3-512withECDSA | The ECDSA signature algorithms as defined in ANSI X9.62.    **Note:** "ECDSA" is an ambiguous name for the "SHA1withECDSA" algorithm and should not be used. The formal name "SHA1withECDSA" should be used instead. |
| NONEwithDSAinP1363Format  SHA1withDSAinP1363Format  SHA224withDSAinP1363Format  SHA256withDSAinP1363Format  SHA384withDSAinP1363Format  SHA512withDSAinP1363Format  SHA3-224withDSAinP1363Format  SHA3-256withDSAinP1363Format  SHA3-384withDSAinP1363Format  SHA3-512withDSAinP1363Format | The DSA signature algorithms as defined in FIPS PUB 186-2, 186-3, and 186-4 with an output as defined in IEEE P1363 format. The format of the Signature bytes for these algorithms is the concatenation of the integers r and s in raw bytes. |
| NONEwithECDSAinP1363Format  SHA1withECDSAinP1363Format  SHA224withECDSAinP1363Format  SHA256withECDSAinP1363Format  SHA384withECDSAinP1363Format  SHA512withECDSAinP1363Format  SHA3-224withECDSAinP1363Format  SHA3-256withECDSAinP1363Format  SHA3-384withECDSAinP1363Format  SHA3-512withECDSAinP1363Format | The ECDSA signature algorithms as defined in ANSI X9.62 and FIPS PUB 186-4 with an output as defined in IEEE P1363 format. The format of the Signature bytes for these algorithms is the concatenation of the integers r and s in raw bytes. |
| <digest>with<encryption> | Use this to form a name for a signature algorithm with a particular message digest (such as MD2 or MD5) and algorithm (such as RSA or DSA), just as was done for the explicitly defined standard names in this section (MD2withRSA, and so on).    For the signature schemes defined in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017), for which the <digest>with<encryption> form is insufficient, <digest>with<encryption>and<mgf> can be used to form a name. Here, <mgf> should be replaced by a mask generation function such as MGF1. Example: **MD5withRSAandMGF1**    For the signature formats defined in IEEE P1363, <digest>with<encryption>in<format>Format can be used to form a name. Example: **SHA1withECDSAinP1363Format** |

## `SSLContext` Algorithms

The algorithm names in this section can be specified when generating
an instance of `SSLContext`.

| Algorithm Name | Description |
| --- | --- |
| SSL | Supports some version of SSL; may support other SSL/TLS versions. |
| SSLv2 | Supports SSL version 2 or later; may support other SSL/TLS versions. |
| SSLv3 | Supports SSL version 3; may support other SSL/TLS versions. |
| TLS | Supports some version of TLS; may support other SSL/TLS versions. |
| TLSv1 | Supports [RFC 2246: TLS version 1.0](https://tools.ietf.org/html/rfc2246); may support other SSL/TLS versions. |
| TLSv1.1 | Supports [RFC 4346: TLS version 1.1](https://tools.ietf.org/html/rfc4346); may support other SSL/TLS versions. |
| TLSv1.2 | Supports [RFC 5246: TLS version 1.2](https://tools.ietf.org/html/rfc5246); may support other SSL/TLS versions. |
| TLSv1.3 | Supports [RFC 8446: TLS version 1.3](https://tools.ietf.org/html/rfc8446); may support other SSL/TLS versions. |
| DTLS | Supports the default provider-dependent versions of DTLS versions. |
| DTLSv1.0 | Supports [RFC 4347: DTLS version 1.0](https://tools.ietf.org/html/rfc4347); may support other DTLS versions. |
| DTLSv1.2 | Supports [RFC 6347: DTLS version 1.2](https://tools.ietf.org/html/rfc6347); may support other DTLS versions. |

## `TrustManagerFactory` Algorithms

The algorithm name in this section can be specified when generating
an instance of `TrustManagerFactory`.

| Algorithm Name | Description |
| --- | --- |
| PKIX | A factory for `X509ExtendedTrustManager` objects that validate certificate chains according to the rules defined by the IETF PKIX working group in [RFC 5280](https://tools.ietf.org/html/rfc5280) or its successor. The `TrustManagerFactory` must support initialization using the class `javax.net.ssl.CertPathTrustManagerParameters`. |

## XML Signature (`XMLSignatureFactory`/`KeyInfoFactory`/`TransformService`) Mechanisms

The mechanism that can be specified when generating an instance of
`XMLSignatureFactory`, `KeyInfoFactory`, or
`TransformService`.

The mechanism identifies the XML processing mechanism that an
implementation uses internally to parse and generate XML signature and
KeyInfo structures. Also, note that each `TransformService`
instance supports a specific transform algorithm in addition to a
mechanism. The standard names for the transform algorithms are defined
in the next section.

| Mechanism | Description |
| --- | --- |
| DOM | The Document Object Model. |

## XML Signature Transform (`TransformService`) Algorithms

The algorithms in this section can be specified when generating an
instance of `TransformService`.

**Note:** The URIs are specified instead of names to be
consistent with the XML Signature standard. API constants have been
defined for each URI, and are listed in parentheses after each URI in
the following table.

| Algorithm URI | Description |
| --- | --- |
| http://www.w3.org/TR/2001/REC-xml-c14n-20010315 (`CanonicalizationMethod.INCLUSIVE`) | The [Canonical XML (without comments)](https://www.w3.org/TR/2001/REC-xml-c14n-20010315) canonicalization algorithm. |
| http://www.w3.org/TR/2001/REC-xml-c14n-20010315#WithComments (`CanonicalizationMethod.INCLUSIVE_WITH_COMMENTS`) | The [Canonical XML with comments](https://www.w3.org/TR/2001/REC-xml-c14n-20010315#WithComments) canonicalization algorithm. |
| http://www.w3.org/2001/10/xml-exc-c14n# (`CanonicalizationMethod.EXCLUSIVE`) | The [Exclusive Canonical XML (without comments)](https://www.w3.org/TR/2002/REC-xml-exc-c14n-20020718/) canonicalization algorithm. |
| http://www.w3.org/2001/10/xml-exc-c14n#WithComments (`CanonicalizationMethod.EXCLUSIVE_WITH_COMMENTS`) | The [Exclusive Canonical XML with comments](https://www.w3.org/TR/2002/REC-xml-exc-c14n-20020718/#WithComments) canonicalization algorithm. |
| http://www.w3.org/2006/12/xml-c14n11 (`CanonicalizationMethod.INCLUSIVE_11`) | The [Canonical XML 1.1 (without comments)](https://www.w3.org/TR/xml-c14n11/) canonicalization algorithm. |
| http://www.w3.org/2006/12/xml-c14n11#WithComments (`CanonicalizationMethod.INCLUSIVE_11_WITH_COMMENTS`) | The [Canonical XML 1.1 with comments](https://www.w3.org/TR/xml-c14n11/#WithComments) canonicalization algorithm. |
| http://www.w3.org/2000/09/xmldsig#base64 (`Transform.BASE64`) | The [Base64](https://www.w3.org/TR/xmldsig-core/#sec-Base-64) transform algorithm. |
| http://www.w3.org/2000/09/xmldsig#enveloped-signature (`Transform.ENVELOPED`) | The [Enveloped Signature](https://www.w3.org/TR/xmldsig-core/#sec-EnvelopedSignature) transform algorithm. |
| http://www.w3.org/TR/1999/REC-xpath-19991116 (`Transform.XPATH`) | The [XPath](https://www.w3.org/TR/xmldsig-core/#sec-XPath) transform algorithm. |
| http://www.w3.org/2002/06/xmldsig-filter2 (`Transform.XPATH2`) | The [XPath Filter 2](https://www.w3.org/TR/2002/REC-xmldsig-filter2-20021108/) transform algorithm. |
| http://www.w3.org/TR/1999/REC-xslt-19991116 (`Transform.XSLT`) | The [XSLT](https://www.w3.org/TR/xmldsig-core/#sec-XSLT) transform algorithm. |

## JSSE Cipher Suite Names

The following table contains the standard JSSE cipher suite names.
Over time, various groups have added additional cipher suites to the [SSL/TLS/DTLS
namespace](https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4).

Some JSSE cipher suite names were defined before TLSv1.0 was
finalized, and were therefore given the `SSL_` prefix. The
names mentioned in the TLS RFCs prefixed with `TLS_` are
functionally equivalent to the JSSE cipher suites prefixed with
`SSL_`.

| Cipher Suite Code | Standard Name (IANA name if different) | Valid for Datagram Transport Layer Protocols | Deprecated (Protocol) | Introduced in (Protocol) | References |
| --- | --- | --- | --- | --- | --- |
| 0x00,0x00 | SSL\_NULL\_WITH\_NULL\_NULL IANA:TLS\_NULL\_WITH\_NULL\_NULL | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x01 | SSL\_RSA\_WITH\_NULL\_MD5 IANA:TLS\_RSA\_WITH\_NULL\_MD5 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x02 | SSL\_RSA\_WITH\_NULL\_SHA IANA:TLS\_RSA\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x03 | SSL\_RSA\_EXPORT\_WITH\_RC4\_40\_MD5 IANA:TLS\_RSA\_EXPORT\_WITH\_RC4\_MD5 | No | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x04 | SSL\_RSA\_WITH\_RC4\_128\_MD5 IANA:TLS\_RSA\_WITH\_RC4\_128\_MD5 | No | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x05 | SSL\_RSA\_WITH\_RC4\_128\_SHA IANA:TLS\_RSA\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x06 | SSL\_RSA\_EXPORT\_WITH\_RC2\_CBC\_40\_MD5 IANA:TLS\_RSA\_EXPORT\_WITH\_RC2\_CBC\_40\_MD5 | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x07 | SSL\_RSA\_WITH\_IDEA\_CBC\_SHA IANA:TLS\_RSA\_WITH\_IDEA\_CBC\_SHA | Yes | TLSv1.2 | N/A | [RFC 5469](https://tools.ietf.org/html/rfc5469) |
| 0x00,0x08 | SSL\_RSA\_EXPORT\_WITH\_DES40\_CBC\_SHA IANA:TLS\_RSA\_EXPORT\_WITH\_DES40\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x09 | SSL\_RSA\_WITH\_DES\_CBC\_SHA IANA:TLS\_RSA\_WITH\_DES\_CBC\_SHA | Yes | TLSv1.2 | N/A | [RFC 5469](https://tools.ietf.org/html/rfc5469) |
| 0x00,0x0A | SSL\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA IANA:TLS\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x0B | SSL\_DH\_DSS\_EXPORT\_WITH\_DES40\_CBC\_SHA IANA:TLS\_DH\_DSS\_EXPORT\_WITH\_DES40\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x0C | SSL\_DH\_DSS\_WITH\_DES\_CBC\_SHA IANA:TLS\_DH\_DSS\_WITH\_DES\_CBC\_SHA | Yes | TLSv1.2 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x0D | SSL\_DH\_DSS\_WITH\_3DES\_EDE\_CBC\_SHA IANA:TLS\_DH\_DSS\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x0E | SSL\_DH\_RSA\_EXPORT\_WITH\_DES40\_CBC\_SHA IANA:TLS\_DH\_RSA\_EXPORT\_WITH\_DES40\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x0F | SSL\_DH\_RSA\_WITH\_DES\_CBC\_SHA IANA:TLS\_DH\_RSA\_WITH\_DES\_CBC\_SHA | Yes | TLSv1.2 | N/A | [RFC 5469](https://tools.ietf.org/html/rfc5469) |
| 0x00,0x10 | SSL\_DH\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA IANA:TLS\_DH\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x11 | SSL\_DHE\_DSS\_EXPORT\_WITH\_DES40\_CBC\_SHA IANA:TLS\_DHE\_DSS\_EXPORT\_WITH\_DES40\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x12 | SSL\_DHE\_DSS\_WITH\_DES\_CBC\_SHA IANA:TLS\_DHE\_DSS\_WITH\_DES\_CBC\_SHA | Yes | TLSv1.2 | N/A | [RFC 5469](https://tools.ietf.org/html/rfc5469) |
| 0x00,0x13 | SSL\_DHE\_DSS\_WITH\_3DES\_EDE\_CBC\_SHA IANA:TLS\_DHE\_DSS\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x14 | SSL\_DHE\_RSA\_EXPORT\_WITH\_DES40\_CBC\_SHA IANA:TLS\_DHE\_RSA\_EXPORT\_WITH\_DES40\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x15 | SSL\_DHE\_RSA\_WITH\_DES\_CBC\_SHA IANA:TLS\_DHE\_RSA\_WITH\_DES\_CBC\_SHA | Yes | TLSv1.2 | N/A | [RFC 5469](https://tools.ietf.org/html/rfc5469) |
| 0x00,0x16 | SSL\_DHE\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA IANA:TLS\_DHE\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x17 | SSL\_DH\_anon\_EXPORT\_WITH\_RC4\_40\_MD5 IANA:TLS\_DH\_anon\_EXPORT\_WITH\_RC4\_40\_MD5 | No | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x18 | SSL\_DH\_anon\_WITH\_RC4\_128\_MD5 IANA:TLS\_DH\_anon\_WITH\_RC4\_128\_MD5 | No | TLSv1.1 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x19 | SSL\_DH\_anon\_EXPORT\_WITH\_DES40\_CBC\_SHA IANA:TLS\_DH\_anon\_EXPORT\_WITH\_DES40\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x1A | SSL\_DH\_anon\_WITH\_DES\_CBC\_SHA IANA:TLS\_DH\_anon\_WITH\_DES\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 4346](https://tools.ietf.org/html/rfc4346) |
| 0x00,0x1B | SSL\_DH\_anon\_WITH\_3DES\_EDE\_CBC\_SHA IANA:TLS\_DH\_anon\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.1 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x1E | TLS\_KRB5\_WITH\_DES\_CBC\_SHA | Yes | TLSv1.2 | TLSv1.0 | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x1F | TLS\_KRB5\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | TLSv1.0 | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x20 | TLS\_KRB5\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | TLSv1.0 | [RFC 2712](https://tools.ietf.org/html/rfc2712) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x21 | TLS\_KRB5\_WITH\_IDEA\_CBC\_SHA | Yes | TLSv1.2 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x22 | TLS\_KRB5\_WITH\_DES\_CBC\_MD5 | Yes | TLSv1.2 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x23 | TLS\_KRB5\_WITH\_3DES\_EDE\_CBC\_MD5 | Yes | TLSv1.3 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x24 | TLS\_KRB5\_WITH\_RC4\_128\_MD5 | No | TLSv1.3 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x25 | TLS\_KRB5\_WITH\_IDEA\_CBC\_MD5 | Yes | TLSv1.2 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x26 | TLS\_KRB5\_EXPORT\_WITH\_DES\_CBC\_40\_SHA | Yes | TLSv1.1 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x27 | TLS\_KRB5\_EXPORT\_WITH\_RC2\_CBC\_40\_SHA | Yes | TLSv1.1 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x28 | TLS\_KRB5\_EXPORT\_WITH\_RC4\_40\_SHA | No | TLSv1.1 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x29 | TLS\_KRB5\_EXPORT\_WITH\_DES\_CBC\_40\_MD5 | Yes | TLSv1.1 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x2A | TLS\_KRB5\_EXPORT\_WITH\_RC2\_CBC\_40\_MD5 | Yes | TLSv1.1 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) |
| 0x00,0x2B | TLS\_KRB5\_EXPORT\_WITH\_RC4\_40\_MD5 | No | TLSv1.1 | N/A | [RFC 2712](https://tools.ietf.org/html/rfc2712) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x2C | TLS\_PSK\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4785](https://tools.ietf.org/html/rfc4785) |
| 0x00,0x2D | TLS\_DHE\_PSK\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4785](https://tools.ietf.org/html/rfc4785) |
| 0x00,0x2E | TLS\_RSA\_PSK\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4785](https://tools.ietf.org/html/rfc4785) |
| 0x00,0x2F | TLS\_RSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x30 | TLS\_DH\_DSS\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x31 | TLS\_DH\_RSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x32 | TLS\_DHE\_DSS\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x33 | TLS\_DHE\_RSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x34 | TLS\_DH\_anon\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x35 | TLS\_RSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x36 | TLS\_DH\_DSS\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x37 | TLS\_DH\_RSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x38 | TLS\_DHE\_DSS\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x39 | TLS\_DHE\_RSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x3A | TLS\_DH\_anon\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x3B | TLS\_RSA\_WITH\_NULL\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x3C | TLS\_RSA\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x3D | TLS\_RSA\_WITH\_AES\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x3E | TLS\_DH\_DSS\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x3F | TLS\_DH\_RSA\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x40 | TLS\_DHE\_DSS\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x41 | TLS\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x42 | TLS\_DH\_DSS\_WITH\_CAMELLIA\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x43 | TLS\_DH\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x44 | TLS\_DHE\_DSS\_WITH\_CAMELLIA\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x45 | TLS\_DHE\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x46 | TLS\_DH\_anon\_WITH\_CAMELLIA\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x67 | TLS\_DHE\_RSA\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x68 | TLS\_DH\_DSS\_WITH\_AES\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x69 | TLS\_DH\_RSA\_WITH\_AES\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x6A | TLS\_DHE\_DSS\_WITH\_AES\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x6B | TLS\_DHE\_RSA\_WITH\_AES\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x6C | TLS\_DH\_anon\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x6D | TLS\_DH\_anon\_WITH\_AES\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5246](https://tools.ietf.org/html/rfc5246) |
| 0x00,0x84 | TLS\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x85 | TLS\_DH\_DSS\_WITH\_CAMELLIA\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x86 | TLS\_DH\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x87 | TLS\_DHE\_DSS\_WITH\_CAMELLIA\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x88 | TLS\_DHE\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x89 | TLS\_DH\_anon\_WITH\_CAMELLIA\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0x8A | TLS\_PSK\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x8B | TLS\_PSK\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x8C | TLS\_PSK\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x8D | TLS\_PSK\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x8E | TLS\_DHE\_PSK\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x8F | TLS\_DHE\_PSK\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x90 | TLS\_DHE\_PSK\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x91 | TLS\_DHE\_PSK\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x92 | TLS\_RSA\_PSK\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0x00,0x93 | TLS\_RSA\_PSK\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x94 | TLS\_RSA\_PSK\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x95 | TLS\_RSA\_PSK\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4279](https://tools.ietf.org/html/rfc4279) |
| 0x00,0x96 | TLS\_RSA\_WITH\_SEED\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4162](https://tools.ietf.org/html/rfc4162) |
| 0x00,0x97 | TLS\_DH\_DSS\_WITH\_SEED\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4162](https://tools.ietf.org/html/rfc4162) |
| 0x00,0x98 | TLS\_DH\_RSA\_WITH\_SEED\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4162](https://tools.ietf.org/html/rfc4162) |
| 0x00,0x99 | TLS\_DHE\_DSS\_WITH\_SEED\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4162](https://tools.ietf.org/html/rfc4162) |
| 0x00,0x9A | TLS\_DHE\_RSA\_WITH\_SEED\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4162](https://tools.ietf.org/html/rfc4162) |
| 0x00,0x9B | TLS\_DH\_anon\_WITH\_SEED\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4162](https://tools.ietf.org/html/rfc4162) |
| 0x00,0x9C | TLS\_RSA\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0x9D | TLS\_RSA\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0x9E | TLS\_DHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0x9F | TLS\_DHE\_RSA\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA0 | TLS\_DH\_RSA\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA1 | TLS\_DH\_RSA\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA2 | TLS\_DHE\_DSS\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA3 | TLS\_DHE\_DSS\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA4 | TLS\_DH\_DSS\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA5 | TLS\_DH\_DSS\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA6 | TLS\_DH\_anon\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA7 | TLS\_DH\_anon\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5288](https://tools.ietf.org/html/rfc5288) |
| 0x00,0xA8 | TLS\_PSK\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xA9 | TLS\_PSK\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xAA | TLS\_DHE\_PSK\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xAB | TLS\_DHE\_PSK\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xAC | TLS\_RSA\_PSK\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xAD | TLS\_RSA\_PSK\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xAE | TLS\_PSK\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xAF | TLS\_PSK\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB0 | TLS\_PSK\_WITH\_NULL\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB1 | TLS\_PSK\_WITH\_NULL\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB2 | TLS\_DHE\_PSK\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB3 | TLS\_DHE\_PSK\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB4 | TLS\_DHE\_PSK\_WITH\_NULL\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB5 | TLS\_DHE\_PSK\_WITH\_NULL\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB6 | TLS\_RSA\_PSK\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB7 | TLS\_RSA\_PSK\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB8 | TLS\_RSA\_PSK\_WITH\_NULL\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xB9 | TLS\_RSA\_PSK\_WITH\_NULL\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5487](https://tools.ietf.org/html/rfc5487) |
| 0x00,0xBA | TLS\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xBB | TLS\_DH\_DSS\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xBC | TLS\_DH\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xBD | TLS\_DHE\_DSS\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xBE | TLS\_DHE\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xBF | TLS\_DH\_anon\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xC0 | TLS\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xC1 | TLS\_DH\_DSS\_WITH\_CAMELLIA\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xC2 | TLS\_DH\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xC3 | TLS\_DHE\_DSS\_WITH\_CAMELLIA\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xC4 | TLS\_DHE\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xC5 | TLS\_DH\_anon\_WITH\_CAMELLIA\_256\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5932](https://tools.ietf.org/html/rfc5932) |
| 0x00,0xFF | TLS\_EMPTY\_RENEGOTIATION\_INFO\_SCSV | Yes | TLSv1.3 | N/A | [RFC 5746](https://tools.ietf.org/html/rfc5746) |
| 0x13,0x01 | TLS\_AES\_128\_GCM\_SHA256 | Yes | N/A | TLSv1.3 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| 0x13,0x02 | TLS\_AES\_256\_GCM\_SHA384 | Yes | N/A | TLSv1.3 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| 0x13,0x03 | TLS\_CHACHA20\_POLY1305\_SHA256 | No | N/A | TLSv1.3 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |
| 0x13,0x04 | TLS\_AES\_128\_CCM\_SHA256 | Yes | N/A | TLSv1.3 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| 0x13,0x05 | TLS\_AES\_128\_CCM\_8\_SHA256 | Yes | N/A | TLSv1.3 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| 0x56,0x00 | TLS\_FALLBACK\_SCSV | Yes | TLSv1.3 | N/A | [RFC 7507](https://tools.ietf.org/html/rfc7507) |
| 0xC0,0x01 | TLS\_ECDH\_ECDSA\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x02 | TLS\_ECDH\_ECDSA\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0xC0,0x03 | TLS\_ECDH\_ECDSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x04 | TLS\_ECDH\_ECDSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x05 | TLS\_ECDH\_ECDSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x06 | TLS\_ECDHE\_ECDSA\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x07 | TLS\_ECDHE\_ECDSA\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0xC0,0x08 | TLS\_ECDHE\_ECDSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x09 | TLS\_ECDHE\_ECDSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x0A | TLS\_ECDHE\_ECDSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x0B | TLS\_ECDH\_RSA\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x0C | TLS\_ECDH\_RSA\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0xC0,0x0D | TLS\_ECDH\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x0E | TLS\_ECDH\_RSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x0F | TLS\_ECDH\_RSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x10 | TLS\_ECDHE\_RSA\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x11 | TLS\_ECDHE\_RSA\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0xC0,0x12 | TLS\_ECDHE\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x13 | TLS\_ECDHE\_RSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x14 | TLS\_ECDHE\_RSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x15 | TLS\_ECDH\_anon\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x16 | TLS\_ECDH\_anon\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0xC0,0x17 | TLS\_ECDH\_anon\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x18 | TLS\_ECDH\_anon\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x19 | TLS\_ECDH\_anon\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 4492](https://tools.ietf.org/html/rfc4492) |
| 0xC0,0x1A | TLS\_SRP\_SHA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x1B | TLS\_SRP\_SHA\_RSA\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x1C | TLS\_SRP\_SHA\_DSS\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x1D | TLS\_SRP\_SHA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x1E | TLS\_SRP\_SHA\_RSA\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x1F | TLS\_SRP\_SHA\_DSS\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x20 | TLS\_SRP\_SHA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x21 | TLS\_SRP\_SHA\_RSA\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x22 | TLS\_SRP\_SHA\_DSS\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5054](https://tools.ietf.org/html/rfc5054) |
| 0xC0,0x23 | TLS\_ECDHE\_ECDSA\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x24 | TLS\_ECDHE\_ECDSA\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x25 | TLS\_ECDH\_ECDSA\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x26 | TLS\_ECDH\_ECDSA\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x27 | TLS\_ECDHE\_RSA\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x28 | TLS\_ECDHE\_RSA\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x29 | TLS\_ECDH\_RSA\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x2A | TLS\_ECDH\_RSA\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x2B | TLS\_ECDHE\_ECDSA\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x2C | TLS\_ECDHE\_ECDSA\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x2D | TLS\_ECDH\_ECDSA\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x2E | TLS\_ECDH\_ECDSA\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x2F | TLS\_ECDHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x30 | TLS\_ECDHE\_RSA\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x31 | TLS\_ECDH\_RSA\_WITH\_AES\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x32 | TLS\_ECDH\_RSA\_WITH\_AES\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 5289](https://tools.ietf.org/html/rfc5289) |
| 0xC0,0x33 | TLS\_ECDHE\_PSK\_WITH\_RC4\_128\_SHA | No | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) [RFC 6347](https://tools.ietf.org/html/rfc6347) |
| 0xC0,0x34 | TLS\_ECDHE\_PSK\_WITH\_3DES\_EDE\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x35 | TLS\_ECDHE\_PSK\_WITH\_AES\_128\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x36 | TLS\_ECDHE\_PSK\_WITH\_AES\_256\_CBC\_SHA | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x37 | TLS\_ECDHE\_PSK\_WITH\_AES\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x38 | TLS\_ECDHE\_PSK\_WITH\_AES\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x39 | TLS\_ECDHE\_PSK\_WITH\_NULL\_SHA | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x3A | TLS\_ECDHE\_PSK\_WITH\_NULL\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x3B | TLS\_ECDHE\_PSK\_WITH\_NULL\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 5489](https://tools.ietf.org/html/rfc5489) |
| 0xC0,0x3C | TLS\_RSA\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x3D | TLS\_RSA\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x3E | TLS\_DH\_DSS\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x3F | TLS\_DH\_DSS\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x40 | TLS\_DH\_RSA\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x41 | TLS\_DH\_RSA\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x42 | TLS\_DHE\_DSS\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x43 | TLS\_DHE\_DSS\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x44 | TLS\_DHE\_RSA\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x45 | TLS\_DHE\_RSA\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x46 | TLS\_DH\_anon\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x47 | TLS\_DH\_anon\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x48 | TLS\_ECDHE\_ECDSA\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x49 | TLS\_ECDHE\_ECDSA\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x4A | TLS\_ECDH\_ECDSA\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x4B | TLS\_ECDH\_ECDSA\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x4C | TLS\_ECDHE\_RSA\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x4D | TLS\_ECDHE\_RSA\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x4E | TLS\_ECDH\_RSA\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x4F | TLS\_ECDH\_RSA\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x50 | TLS\_RSA\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x51 | TLS\_RSA\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x52 | TLS\_DHE\_RSA\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x53 | TLS\_DHE\_RSA\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x54 | TLS\_DH\_RSA\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x55 | TLS\_DH\_RSA\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x56 | TLS\_DHE\_DSS\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x57 | TLS\_DHE\_DSS\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x58 | TLS\_DH\_DSS\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x59 | TLS\_DH\_DSS\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x5A | TLS\_DH\_anon\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x5B | TLS\_DH\_anon\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x5C | TLS\_ECDHE\_ECDSA\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x5D | TLS\_ECDHE\_ECDSA\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x5E | TLS\_ECDH\_ECDSA\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x5F | TLS\_ECDH\_ECDSA\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x60 | TLS\_ECDHE\_RSA\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x61 | TLS\_ECDHE\_RSA\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x62 | TLS\_ECDH\_RSA\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x63 | TLS\_ECDH\_RSA\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x64 | TLS\_PSK\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x65 | TLS\_PSK\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x66 | TLS\_DHE\_PSK\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x67 | TLS\_DHE\_PSK\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x68 | TLS\_RSA\_PSK\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x69 | TLS\_RSA\_PSK\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x6A | TLS\_PSK\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x6B | TLS\_PSK\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x6C | TLS\_DHE\_PSK\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x6D | TLS\_DHE\_PSK\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x6E | TLS\_RSA\_PSK\_WITH\_ARIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x6F | TLS\_RSA\_PSK\_WITH\_ARIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x70 | TLS\_ECDHE\_PSK\_WITH\_ARIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x71 | TLS\_ECDHE\_PSK\_WITH\_ARIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6209](https://tools.ietf.org/html/rfc6209) |
| 0xC0,0x72 | TLS\_ECDHE\_ECDSA\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x73 | TLS\_ECDHE\_ECDSA\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x74 | TLS\_ECDH\_ECDSA\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x75 | TLS\_ECDH\_ECDSA\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x76 | TLS\_ECDHE\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x77 | TLS\_ECDHE\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x78 | TLS\_ECDH\_RSA\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x79 | TLS\_ECDH\_RSA\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x7A | TLS\_RSA\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x7B | TLS\_RSA\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x7C | TLS\_DHE\_RSA\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x7D | TLS\_DHE\_RSA\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x7E | TLS\_DH\_RSA\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x7F | TLS\_DH\_RSA\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x80 | TLS\_DHE\_DSS\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x81 | TLS\_DHE\_DSS\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x82 | TLS\_DH\_DSS\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x83 | TLS\_DH\_DSS\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x84 | TLS\_DH\_anon\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x85 | TLS\_DH\_anon\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x86 | TLS\_ECDHE\_ECDSA\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x87 | TLS\_ECDHE\_ECDSA\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x88 | TLS\_ECDH\_ECDSA\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x89 | TLS\_ECDH\_ECDSA\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x8A | TLS\_ECDHE\_RSA\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x8B | TLS\_ECDHE\_RSA\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x8C | TLS\_ECDH\_RSA\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x8D | TLS\_ECDH\_RSA\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x8E | TLS\_PSK\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x8F | TLS\_PSK\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x90 | TLS\_DHE\_PSK\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x91 | TLS\_DHE\_PSK\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x92 | TLS\_RSA\_PSK\_WITH\_CAMELLIA\_128\_GCM\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x93 | TLS\_RSA\_PSK\_WITH\_CAMELLIA\_256\_GCM\_SHA384 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x94 | TLS\_PSK\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x95 | TLS\_PSK\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x96 | TLS\_DHE\_PSK\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x97 | TLS\_DHE\_PSK\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x98 | TLS\_RSA\_PSK\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x99 | TLS\_RSA\_PSK\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x9A | TLS\_ECDHE\_PSK\_WITH\_CAMELLIA\_128\_CBC\_SHA256 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x9B | TLS\_ECDHE\_PSK\_WITH\_CAMELLIA\_256\_CBC\_SHA384 | Yes | TLSv1.3 | N/A | [RFC 6367](https://tools.ietf.org/html/rfc6367) |
| 0xC0,0x9C | TLS\_RSA\_WITH\_AES\_128\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0x9D | TLS\_RSA\_WITH\_AES\_256\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0x9E | TLS\_DHE\_RSA\_WITH\_AES\_128\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0x9F | TLS\_DHE\_RSA\_WITH\_AES\_256\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA0 | TLS\_RSA\_WITH\_AES\_128\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA1 | TLS\_RSA\_WITH\_AES\_256\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA2 | TLS\_DHE\_RSA\_WITH\_AES\_128\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA3 | TLS\_DHE\_RSA\_WITH\_AES\_256\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA4 | TLS\_PSK\_WITH\_AES\_128\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA5 | TLS\_PSK\_WITH\_AES\_256\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA6 | TLS\_DHE\_PSK\_WITH\_AES\_128\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA7 | TLS\_DHE\_PSK\_WITH\_AES\_256\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA8 | TLS\_PSK\_WITH\_AES\_128\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xA9 | TLS\_PSK\_WITH\_AES\_256\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xAA | TLS\_DHE\_PSK\_WITH\_AES\_128\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xAB | TLS\_DHE\_PSK\_WITH\_AES\_256\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 6655](https://tools.ietf.org/html/rfc6655) |
| 0xC0,0xAC | TLS\_ECDHE\_ECDSA\_WITH\_AES\_128\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 7251](https://tools.ietf.org/html/rfc7251) |
| 0xC0,0xAD | TLS\_ECDHE\_ECDSA\_WITH\_AES\_256\_CCM | Yes | TLSv1.3 | TLSv1.2 | [RFC 7251](https://tools.ietf.org/html/rfc7251) |
| 0xC0,0xAE | TLS\_ECDHE\_ECDSA\_WITH\_AES\_128\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7251](https://tools.ietf.org/html/rfc7251) |
| 0xC0,0xAF | TLS\_ECDHE\_ECDSA\_WITH\_AES\_256\_CCM\_8 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7251](https://tools.ietf.org/html/rfc7251) |
| 0xCC,0xA8 | TLS\_ECDHE\_RSA\_WITH\_CHACHA20\_POLY1305\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |
| 0xCC,0xA9 | TLS\_ECDHE\_ECDSA\_WITH\_CHACHA20\_POLY1305\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |
| 0xCC,0xAA | TLS\_DHE\_RSA\_WITH\_CHACHA20\_POLY1305\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |
| 0xCC,0xAB | TLS\_PSK\_WITH\_CHACHA20\_POLY1305\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |
| 0xCC,0xAC | TLS\_ECDHE\_PSK\_WITH\_CHACHA20\_POLY1305\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |
| 0xCC,0xAD | TLS\_DHE\_PSK\_WITH\_CHACHA20\_POLY1305\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |
| 0xCC,0xAE | TLS\_RSA\_PSK\_WITH\_CHACHA20\_POLY1305\_SHA256 | Yes | TLSv1.3 | TLSv1.2 | [RFC 7905](https://tools.ietf.org/html/rfc7905) |

## Additional JSSE Standard Names

### Key Types

The `keyType` parameter passed to the
`chooseClientAlias`, `chooseServerAlias`,
`getClientAliases`, and `getServerAliases` methods
of `X509KeyManager` specifies the public key types.

Each row of the table that follows lists the standard name that
should be used for `keyType`, given the specified certificate
type.

Standard Names for a Key Type

| Name | Certificate Type |
| --- | --- |
| RSA | RSA |
| DSA | DSA |
| DH\_RSA | Diffie-Hellman with RSA signature |
| DH\_DSA | Diffie-Hellman with DSA signature |
| EC | Elliptic Curve |
| EC\_EC | Elliptic Curve with ECDSA signature |
| EC\_RSA | Elliptic Curve with RSA signature |
| RSASSA-PSS | RSASSA-PSS |
| EdDSA | EdDSA (Ed25519 and Ed448) |

### Protocols

The `protocols` parameter passed to the
`setEnabledProtocols` method of `SSLSocket` and
`SSLEngine` specifies the protocol versions to be enabled for
use on the connection. The table that follows lists the standard names
that can be passed to the `setEnabledProtocols` method or
that may be returned by the `getSupportedProtocols` and
`getEnabledProtocols` methods of `SSLSocket` and
`SSLEngine`.

These names also apply to the `protocol` parameter
returned from the `getProtocol` method of
`SSLSession`, and the `protocols` parameter passed
to the `setProtocols` method or that may be returned by the
`getProtocols` method of `SSLParameters`.

Standard Names for a Protocol

| Name | Protocol |
| --- | --- |
| SSLv2 | SSL version 2 protocol |
| SSLv3 | SSL version 3 protocol |
| TLSv1 | TLS version 1.0 protocol (defined in [RFC 2246](https://tools.ietf.org/html/rfc2246)) |
| TLSv1.1 | TLS version 1.1 protocol (defined in [RFC 4346](https://tools.ietf.org/html/rfc4346)) |
| TLSv1.2 | TLS version 1.2 protocol (defined in [RFC 5246](https://tools.ietf.org/html/rfc5246)) |
| TLSv1.3 | TLS version 1.3 protocol (defined in [RFC 8446](https://tools.ietf.org/html/rfc8446)) |
| DTLSv1.0 | DTLS version 1.0 protocol (defined in [RFC 4347](https://tools.ietf.org/html/rfc4347)) |
| DTLSv1.2 | DTLS version 1.2 protocol (defined in [RFC 6347](https://tools.ietf.org/html/rfc6347)) |
| SSLv2Hello | Currently, the SSLv3, TLSv1, and TLSv1.1 protocols allow you to send SSLv3, TLSv1, and TLSv1.1 hellos encapsulated in an SSLv2 format hello. For more details on the reasons for allowing this compatibility in these protocols, see Appendix E in the appropriate RFCs (previously listed).    **Note:** Some SSL/TLS servers do not support the v2 hello format and require that client hellos conform to the SSLv3 or TLSv1 client hello formats.    The SSLv2Hello option controls the SSLv2 encapsulation. If SSLv2Hello is disabled on the client, then all outgoing messages will conform to the SSLv3/TLSv1 client hello format. If SSLv2Hello is disabled on the server, then all incoming messages must conform to the SSLv3/TLSv1 client hello format. |

### Authentication Types

The `authType` parameter passed to the
`checkClientTrusted` and `checkServerTrusted`
methods of `X509TrustManager` indicates the authentication
type. The table that follows specifies what standard names should be
used for the client or server certificate chains.

Standard Names for Client or Server Certificate Chain

| Client or Server Certificate Chain | Authentication Type Standard Name |
| --- | --- |
| Client | Determined by the actual certificate used. For instance, if RSAPublicKey is used, the `authType` should be "RSA". |
| Server | The key exchange algorithm portion of the cipher suites represented as a String, such as "RSA" or "DHE\_DSS".    **Note:** For some exportable cipher suites, the key exchange algorithm is determined at runtime during the handshake.    For instance, for TLS\_RSA\_EXPORT\_WITH\_RC4\_40\_MD5, the `authType` should be "RSA\_EXPORT" when an ephemeral RSA key is used for the key exchange, and "RSA" when the key from the server certificate is used. Or it can take the value "UNKNOWN". |

### Endpoint Identification Algorithms

The endpoint identification algorithm indicates the endpoint
identification or verification procedures during SSL/TLS/DTLS
handshaking. The algorithm name can be passed to the
`setEndpointIdentificationAlgorithm` method of
`javax.net.ssl.SSLParameters`.

The following table shows the standard endpoint identification
names.

Endpoint Identification Algorithm Name

| Endpoint Identification Algorithm Name | Specification |
| --- | --- |
| HTTPS | [RFC 2818](https://tools.ietf.org/html/rfc2818) |
| LDAPS | [RFC 2830](https://tools.ietf.org/html/rfc2830) |

### Signature Schemes

The following table contains the standard signature scheme names,
which are the algorithms used in the digital signatures of TLS
connections and are also defined in the [SignatureScheme
section](https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-signaturescheme) of the IANA TLS Registry.

| Signature Scheme | Specification |
| --- | --- |
| ecdsa\_secp256r1\_sha256 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| ecdsa\_secp384r1\_sha384 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| ecdsa\_secp521r1\_sha512 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| ecdsa\_sha1 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| ed25519 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| ed448 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pkcs1\_sha1 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pkcs1\_sha256 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pkcs1\_sha384 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pkcs1\_sha512 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pss\_pss\_sha256 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pss\_pss\_sha384 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pss\_pss\_sha512 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pss\_rsae\_sha256 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pss\_rsae\_sha384 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |
| rsa\_pss\_rsae\_sha512 | [RFC 8446](https://tools.ietf.org/html/rfc8446) |

### Named Groups

The following table contains the standard group names, which are the
named groups used in key exchange algorithms of TLS connections and are
also defined in the [Supported
Groups section](https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-8) of the IANA TLS Registry.

| Name |  |
| --- | --- |
| secp256r1  secp384r1  secp521r1 | The NIST elliptic curves as specified in [RFC 8422](https://tools.ietf.org/html/rfc8422). |
| x25519  x448 | The elliptic curves as specified in [RFC 8446](https://tools.ietf.org/html/rfc8446) and [RFC 8442](https://tools.ietf.org/html/rfc8422). |
| ffdhe2048  ffdhe3072  ffdhe4096  ffdhe6144  ffdhe8192 | The Finite Field Diffie-Hellman Ephemeral (FFDHE) groups as specified in [RFC 7919](https://tools.ietf.org/html/rfc7919). |

## Security Algorithm Specification

This section specifies details concerning some of the algorithms
defined in this document. Any provider supplying an implementation of
the listed algorithms must comply with the specifications in this
section.

### Specification Template

The following table shows the fields of the algorithm
specifications.

| Field | Description |
| --- | --- |
| Name | The name by which the algorithm is known. This is the name passed to the `getInstance` method (when requesting the algorithm), and returned by the `getAlgorithm` method to determine the name of an existing algorithm object. These methods are in the relevant engine classes: `Signature`, `MessageDigest`, `KeyPairGenerator`, and `AlgorithmParameterGenerator` . |
| Type | The type of algorithm: `Signature`, `MessageDigest`, `KeyPairGenerator`, or `AlgorithmParameterGenerator`. |
| Description | General notes about the algorithm, including any standards implemented by the algorithm, applicable patents, and so on. |
| `KeyPair` Algorithm (*optional*) | The `KeyPair` algorithm for this algorithm. |
| Keysize (*optional*) | For a keyed algorithm or key generation algorithm: the valid keysizes. |
| Size (*optional*) | For an algorithm parameter generation algorithm: the valid "sizes" for algorithm parameter generation. |
| Parameter Defaults (*optional*) | For a key generation algorithm: the default parameter values. |
| `Signature` Format (*optional*) | For a `Signature` algorithm, the format of the signature, that is, the input and output of the verify and sign methods, respectively. |

### Algorithm Specifications

SHA-1 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | SHA-1 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf). The output of this algorithm is a 160-bit digest. |

SHA-224 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | SHA-224 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf). The output of this algorithm is a 224-bit digest. |

SHA-256 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | SHA-256 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf). The output of this algorithm is a 256-bit digest. |

SHA-384 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | SHA-384 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf). The output of this algorithm is a 384-bit digest. |

SHA-512 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | SHA-512 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf). The output of this algorithm is a 512-bit digest. |

SHA-512/224 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | SHA-512/224 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf). The output of this algorithm is a 224-bit digest. |

SHA-512/256 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | SHA-512/256 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS 180-4](https://csrc.nist.gov/publications/fips/fips180-4/fips-180-4.pdf). The output of this algorithm is a 256-bit digest. |

SHA3-224 Message Digest Algorithms

| Field | Description |
| --- | --- |
| Name | SHA3-224 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS PUB 202](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf). The output of this algorithm is a 224-bit digest. |

SHA3-256 Message Digest Algorithms

| Field | Description |
| --- | --- |
| Name | SHA3-256 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS PUB 202](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf). The output of this algorithm is a 256-bit digest. |

SHA3-384 Message Digest Algorithms

| Field | Description |
| --- | --- |
| Name | SHA3-384 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS PUB 202](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf). The output of this algorithm is a 384-bit digest. |

SHA3-512 Message Digest Algorithms

| Field | Description |
| --- | --- |
| Name | SHA3-512 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [FIPS PUB 202](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf). The output of this algorithm is a 512-bit digest. |

MD2 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | MD2 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [RFC 1319](https://tools.ietf.org/html/rfc1319). The output of this algorithm is a 128-bit digest. |

MD5 Message Digest Algorithm

| Field | Description |
| --- | --- |
| Name | MD5 |
| Type | `MessageDigest` |
| Description | The message digest algorithm as defined in [RFC 1321](https://tools.ietf.org/html/rfc1321). The output of this algorithm is a 128-bit digest. |

The Digital Signature Algorithms, with SHA-1 or SHA-2

| Field | Description |
| --- | --- |
| Name | SHA1withDSA, SHA224withDSA, SHA256withDSA, SHA384withDSA, and SHA512withDSA |
| Type | `Signature` |
| Description | The signature algorithm described in [NIST FIPS 186-3](https://csrc.nist.gov/publications/fips/fips186-3/fips_186-3.pdf), using DSA with the SHA-1, SHA-224, SHA-256, SHA-384, and SHA-512 message digest algorithms. |
| `KeyPair` Algorithm | DSA |
| `Signature` Format | ASN.1 sequence of two INTEGER values: `r` and `s`, in that order:    `SEQUENCE { r INTEGER, s INTEGER }` |

RSA-based Signature Algorithms, with MD2, MD5, SHA-1, or
SHA-2

| Field | Description |
| --- | --- |
| Names | MD2withRSA, MD5withRSA, SHA1withRSA, SHA224withRSA, SHA256withRSA, SHA384withRSA, SHA512withRSA, SHA512/224withRSA, SHA512/256withRSA |
| Type | `Signature` |
| Description | These are the signature algorithms that use the MD2, MD5, SHA-1, SHA-224, SHA-256, SHA-384, and SHA-512 message digest algorithms (respectively) with RSA encryption. |
| `KeyPair` Algorithm | RSA |
| `Signature` Format | DER-encoded PKCS #1 block as defined in [RSA Laboratories, PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). The data encrypted is the digest of the data signed. |

RSASSA-PSS-based Signature Algorithms

| Field | Description |
| --- | --- |
| Names | RSASSA-PSS |
| Type | `Signature` |
| Description | This signature algorithm requires PSS parameters to be explicitly supplied before data can be processed. |
| `KeyPair` Algorithm | RSA or RSASSA-PSS |
| `Signature` Format | DER-encoded PKCS1 block as defined in [RSA Laboratories, PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). The data encrypted is the digest of the data signed. |

DSA KeyPair Generation Algorithm

| Field | Description |
| --- | --- |
| Name | DSA |
| Type | `KeyPairGenerator` |
| Description | This algorithm is the key pair generation algorithm described in [NIST FIPS 186](https://csrc.nist.gov/publications/PubsFIPS.html) for DSA. |
| Keysize | The length, in bits, of the modulus `p`. This must be a multiple of 64, ranging from 512 to 1024 (inclusive), 2048, or 3072. |
| Parameter Defaults | See below for the parameter values. |

The following are the parameter values for keysizes of 512, 768, and
1024 bits:

**512-bit Key Parameters**

```
 SEED =
     b869c82b 35d70e1b 1ff91b28 e37a62ec dc34409b
 counter = 123
 p =
     fca682ce 8e12caba 26efccf7 110e526d b078b05e decbcd1e b4a208f3
     ae1617ae 01f35b91 a47e6df6 3413c5e1 2ed0899b cd132acd 50d99151
     bdc43ee7 37592e17
 q =
     962eddcc 369cba8e bb260ee6 b6a126d9 346e38c5
 g =
     678471b2 7a9cf44e e91a49c5 147db1a9 aaf244f0 5a434d64 86931d2d
     14271b9e 35030b71 fd73da17 9069b32e 2935630e 1c206235 4d0da20a
     6c416e50 be794ca4
```

**768-bit key parameters**

```
 SEED =
     77d0f8c4 dad15eb8 c4f2f8d6 726cefd9 6d5bb399
 counter = 263
 p =
     e9e64259 9d355f37 c97ffd35 67120b8e 25c9cd43 e927b3a9 670fbec5
     d8901419 22d2c3b3 ad248009 3799869d 1e846aab 49fab0ad 26d2ce6a
     22219d47 0bce7d77 7d4a21fb e9c270b5 7f607002 f3cef839 3694cf45
     ee3688c1 1a8c56ab 127a3daf
 q =
     9cdbd84c 9f1ac2f3 8d0f80f4 2ab952e7 338bf511
 g =
     30470ad5 a005fb14 ce2d9dcd 87e38bc7 d1b1c5fa cbaecbe9 5f190aa7
     a31d23c4 dbbcbe06 17454440 1a5b2c02 0965d8c2 bd2171d3 66844577
     1f74ba08 4d2029d8 3c1c1585 47f3a9f1 a2715be2 3d51ae4d 3e5a1f6a
     7064f316 933a346d 3f529252
```

**1024-bit key parameters**

```
 SEED =
     8d515589 4229d5e6 89ee01e6 018a237e 2cae64cd
 counter = 92
 p =
     fd7f5381 1d751229 52df4a9c 2eece4e7 f611b752 3cef4400 c31e3f80
     b6512669 455d4022 51fb593d 8d58fabf c5f5ba30 f6cb9b55 6cd7813b
     801d346f f26660b7 6b9950a5 a49f9fe8 047b1022 c24fbba9 d7feb7c6
     1bf83b57 e7c6a8a6 150f04fb 83f6d3c5 1ec30235 54135a16 9132f675
     f3ae2b61 d72aeff2 2203199d d14801c7
 q =
     9760508f 15230bcc b292b982 a2eb840b f0581cf5
 g =
     f7e1a085 d69b3dde cbbcab5c 36b857b9 7994afbb fa3aea82 f9574c0b
     3d078267 5159578e bad4594f e6710710 8180b449 167123e8 4c281613
     b7cf0932 8cc8a6e1 3c167a8b 547c8d28 e0a3ae1e 2bb3a675 916ea37f
     0bfa2135 62f1fb62 7a01243b cca4f1be a8519089 a883dfe1 5ae59f06
     928b665e 807b5525 64014c3b fecf492a
```

The following are the default values for larger DSA key sizes
identified by (L,N) pairs:

**(L,N) = (2048, 256)**

```
 SEED =
     b0b44176 01b59cbc 9d8ac8f9 35cadaec 4f5fbb2f 23785609 ae466748
     d9b5a536
 counter = 497
 p =
     95475cf5 d93e596c 3fcd1d90 2add02f4 27f5f3c7 210313bb 45fb4d5b
     b2e5fe1c bd678cd4 bbdd84c9 836be1f3 1c077772 5aeb6c2f c38b85f4
     8076fa76 bcd8146c c89a6fb2 f706dd71 9898c208 3dc8d896 f84062e2
     c9c94d13 7b054a8d 8096adb8 d5195239 8eeca852 a0af12df 83e475aa
     65d4ec0c 38a9560d 5661186f f98b9fc9 eb60eee8 b030376b 236bc73b
     e3acdbd7 4fd61c1d 2475fa30 77b8f080 467881ff 7e1ca56f ee066d79
     506ade51 edbb5443 a563927d bc4ba520 08674617 5c888592 5ebc64c6
     14790677 3496990c b714ec66 7304e261 faee33b3 cbdf008e 0c3fa906
     50d97d39 09c9275b f4ac86ff cb3d03e6 dfc8ada5 934242dd 6d3bcca2
     a406cb0b
 q =
     f8183668 ba5fc5bb 06b5981e 6d8b795d 30b8978d 43ca0ec5 72e37e09
     939a9773
 g =
     42debb9d a5b3d88c c956e087 87ec3f3a 09bba5f4 8b889a74 aaf53174
     aa0fbe7e 3c5b8fcd 7a53bef5 63b0e985 60328960 a9517f40 14d3325f
     c7962bf1 e049370d 76d1314a 76137e79 2f3f0db8 59d095e4 a5b93202
     4f079ecf 2ef09c79 7452b077 0e135078 2ed57ddf 794979dc ef23cb96
     f1830619 65c4ebc9 3c9c71c5 6b925955 a75f94cc cf1449ac 43d586d0
     beee4325 1b0b2287 349d68de 0d144403 f13e802f 4146d882 e057af19
     b6f6275c 6676c8fa 0e3ca271 3a3257fd 1b27d063 9f695e34 7d8d1cf9
     ac819a26 ca9b04cb 0eb9b7b0 35988d15 bbac6521 2a55239c fc7e58fa
     e38d7250 ab9991ff bc971340 25fe8ce0 4c4399ad 96569be9 1a546f49
     78693c7a
```

**(L,N) = (2048, 224)**

```
 SEED =
     58423608 0cfa43c0 9b023541 35f4cc51 98a19efa da08bd86 6d601ba4
 counter = 2666
 p =
     8f7935d9 b9aae9bf abed887a cf4951b6 f32ec59e 3baf3718 e8eac496
     1f3efd36 06e74351 a9c41833 39b809e7 c2ae1c53 9ba7475b 85d011ad
     b8b47987 75498469 5cac0e8f 14b33608 28a22ffa 27110a3d 62a99345
     3409a0fe 696c4658 f84bdd20 819c3709 a01057b1 95adcd00 233dba54
     84b6291f 9d648ef8 83448677 979cec04 b434a6ac 2e75e998 5de23db0
     292fc111 8c9ffa9d 8181e733 8db792b7 30d7b9e3 49592f68 09987215
     3915ea3d 6b8b4653 c633458f 803b32a4 c2e0f272 90256e4e 3f8a3b08
     38a1c450 e4e18c1a 29a37ddf 5ea143de 4b66ff04 903ed5cf 1623e158
     d487c608 e97f211c d81dca23 cb6e3807 65f822e3 42be484c 05763939
     601cd667
 q =
     baf696a6 8578f7df dee7fa67 c977c785 ef32b233 bae580c0 bcd5695d
 g =
     16a65c58 20485070 4e7502a3 9757040d 34da3a34 78c154d4 e4a5c02d
     242ee04f 96e61e4b d0904abd ac8f37ee b1e09f31 82d23c90 43cb642f
     88004160 edf9ca09 b32076a7 9c32a627 f2473e91 879ba2c4 e744bd20
     81544cb5 5b802c36 8d1fa83e d489e94e 0fa0688e 32428a5c 78c478c6
     8d0527b7 1c9a3abb 0b0be12c 44689639 e7d3ce74 db101a65 aa2b87f6
     4c6826db 3ec72f4b 5599834b b4edb02f 7c90e9a4 96d3a55d 535bebfc
     45d4f619 f63f3ded bb873925 c2f224e0 7731296d a887ec1e 4748f87e
     fb5fdeb7 5484316b 2232dee5 53ddaf02 112b0d1f 02da3097 3224fe27
     aeda8b9d 4b2922d9 ba8be39e d9e103a6 3c52810b c688b7e2 ed4316e1
     ef17dbde
```

RSA KeyPair Generation Algorithm

| Field | Description |
| --- | --- |
| Names | RSA |
| Type | `KeyPairGenerator` |
| Description | This algorithm is the key pair generation algorithm described in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). |
| Strength | The length, in bits, of the modulus `n`. This must be a multiple of 8 that is greater than or equal to 512 |

RSASSA-PSS KeyPair Generation Algorithm

| Field | Description |
| --- | --- |
| Names | RSASSA-PSS |
| Type | `KeyPairGenerator` |
| Description | This algorithm is the key pair generation algorithm described in [PKCS #1 v2.2](https://tools.ietf.org/html/rfc8017). |
| Strength | The length, in bits, of the modulus `n`. This must be a multiple of 8 that is greater than or equal to 512 |

DSA Parameter Generation Algorithm

| Field | Description |
| --- | --- |
| Names | DSA |
| Type | `AlgorithmParameterGenerator` |
| Description | This algorithm is the parameter generation algorithm described in [NIST FIPS 186](https://csrc.nist.gov/publications/PubsFIPS.html) for DSA. |
| Strength | The length, in bits, of the modulus `p`. This must be a multiple of 64, ranging from from 512 to 1024 (inclusive), 2048, or 3072.  Alternatively, generate DSA parameters with the [DSAGenParameterSpec](../../api/java.base/java/security/spec/DSAGenParameterSpec.html) class. Note that this class supports the latest version of DSA standard, [FIPS PUB 186-3](https://csrc.nist.gov/publications/fips/fips186-3/fips_186-3.pdf), and only allows certain length of prime P and Q to be used. Valid sizes for length of prime P and sub-prime Q in bits are as follows:    (1024, 160)  (2048, 224)  (2048, 256)  (3072, 256) |

## Security Algorithm Implementation Requirements

This section defines the security algorithm requirements for Java SE
implementations. The security algorithm requirements are intended to
improve the interoperability of Java SE implementations and applications
that use these algorithms.

**Note:** The requirements in this section are
**not** a measure of the strength or security of the
algorithm. For example, recent advances in cryptanalysis have found
weaknesses in the strength of the DESede (Triple DES) cipher algorithm.
It is your responsibility to determine whether the algorithm meets the
security requirements of your application.

Every implementation of this version of the Java SE platform must
support the specified algorithms in the table that follows. These
requirements do not apply to 3rd party providers. Consult the release
documentation for your implementation to see if any other algorithms are
supported.

| Class | Algorithm Name(s) |
| --- | --- |
| `AlgorithmParameterGenerator`  Implementations must support the key sizes in parentheses. | DiffieHellman (1024, 2048)  DSA (1024, 2048) |
| `AlgorithmParameters`  For the "EC" algorithm, implementations must support the curves in parentheses. For the "RSASSA-PSS" algorithm, implementations must support the parameters in parentheses. | AES  ChaCha20-Poly1305  DESede  DiffieHellman  DSA  EC (secp256r1, secp384r1)  RSASSA-PSS (MGF1 mask generation function and SHA-256 or SHA-384 hash algorithms) |
| `CertificateFactory` | X.509 |
| `CertPath` Encoding | PKCS7  PkiPath |
| `CertPathBuilder` | PKIX |
| `CertPathValidator` | PKIX |
| `CertStore` | Collection |
| `Cipher`  Implementations must support the key sizes in parentheses. | AES/CBC/NoPadding (128)  AES/CBC/PKCS5Padding (128)  AES/ECB/NoPadding (128)  AES/ECB/PKCS5Padding (128)  AES/GCM/NoPadding (128, 256)  ChaCha20-Poly1305  DESede/CBC/NoPadding (168)  DESede/CBC/PKCS5Padding (168)  DESede/ECB/NoPadding (168)  DESede/ECB/PKCS5Padding (168)  RSA/ECB/PKCS1Padding (1024, 2048)  RSA/ECB/OAEPWithSHA-1AndMGF1Padding (1024, 2048)  RSA/ECB/OAEPWithSHA-256AndMGF1Padding (1024, 2048) |
| `Configuration` [[1]](#footnote-1) |  |
| `KeyAgreement`  For the "ECDH" algorithm, implementations must support the curves in parentheses. | DiffieHellman  ECDH (secp256r1, secp384r1)  X25519 |
| `KeyFactory` | DiffieHellman  DSA  EC  RSA  RSASSA-PSS  X25519 |
| `KeyGenerator`  Implementations must support the key sizes in parentheses. | AES (128, 256)  ChaCha20  DESede (168)  HmacSHA1  HmacSHA256 |
| `KeyPairGenerator`  For the "EC" algorithm, implementations must support the curves in parentheses. For other algorithms, implementations must support the key sizes in parentheses. | DiffieHellman (1024, 2048, 3072, 4096)  DSA (1024, 2048)  EC (secp256r1, secp384r1)  RSA (1024, 2048, 3072, 4096)  RSASSA-PSS (2048, 3072, 4096)  X25519 |
| `KeyStore` | PKCS12 |
| `Mac` | HmacSHA1  HmacSHA256 |
| `MessageDigest` | SHA-1  SHA-256  SHA-384 |
| `SecretKeyFactory` | DESede |
| `SecureRandom` [[1]](#footnote-1) |  |
| `Signature`  For the "RSASSA-PSS" algorithm, implementations must support the parameters in parentheses. For the "SHA256withECDSA" and "SHA384withECDSA" algorithms, implementations must support the curves in parentheses. | RSASSA-PSS (MGF1 mask generation function and SHA-256 or SHA-384 hash algorithms)  SHA1withDSA  SHA256withDSA  SHA256withECDSA (secp256r1)  SHA384withECDSA (secp384r1)  SHA1withRSA  SHA256withRSA  SHA384withRSA |
| `SSLContext` | TLSv1.2  TLSv1.3 |
| `TrustManagerFactory` | PKIX |

**[1]** No specific
`Configuration` type or `SecureRandom` algorithm
is required; however, an implementation-specific default must be
provided.

### XML Signature Algorithms

Every implementation of this version of the Java SE platform must
support the specified XML Signature algorithms in the table that
follows. These requirements do not apply to 3rd party providers. Consult
the release documentation for your implementation to see if any other
algorithms are supported.

| Class | Algorithm Name(s) |
| --- | --- |
| `TransformService` | http://www.w3.org/2001/10/xml-exc-c14n# (`CanonicalizationMethod.EXCLUSIVE`)  http://www.w3.org/TR/2001/REC-xml-c14n-20010315 (`CanonicalizationMethod.INCLUSIVE`)  http://www.w3.org/2000/09/xmldsig#base64 (`Transform.BASE64`)  http://www.w3.org/2000/09/xmldsig#enveloped-signature (`Transform.ENVELOPED`) |
| `XMLSignatureFactory` | DOM |

---

[Copyright](../../legal/copyright.html) © 1993, 2025, Oracle and/or its affiliates, 500 Oracle Parkway, Redwood Shores, CA 94065 USA.
All rights reserved. Use is subject to [license terms](https://www.oracle.com/java/javase/terms/license/java25speclicense.html) and the [documentation redistribution policy](https://www.oracle.com/technetwork/java/redist-137594.html).

Scripting on this page tracks web page traffic, but does not change the content in any way.
