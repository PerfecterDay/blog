# Java 加密
{docsify-updated}

> https://www.ibm.com/docs/en/sdk-java-technology/8?topic=class-creating-cipher-object  
> https://docs.oracle.com/en/java/javase/17/security/terms-and-definitions.html

<center><img src="jce.png" width="40%"></center>

## 加密相关的概念

### 加密与解密
加密是获取数据（称为明文）和一个短字符串（密钥），并生成对不知道密钥的第三方毫无意义的数据（密文）的过程。 解密则是一个相反的过程：即获取密文和一串简短的密钥，然后生成明文。

### 基于密码的加密 (Password-Based Encryption)
基于密码的加密（PBE）从密码中提取加密密钥。 为了让攻击者在从密码到密钥的过程中耗费大量时间，大多数 PBE 实现都会在创建密钥时混入一个随机数（称为 "盐"）。

### Cipher
加密和解密是通过 `Cipher` 完成的。  `Cipher` 是能够根据加密方案（算法）进行加密和解密的对象。

### Key Agreement
密钥协议是一种协议，通过它，两方或多方可以建立相同的加密密钥，而无需交换任何秘密信息。

### Message Authentication Code
信息验证码 (Message Authentication Code-MAC) 提供了一种基于密钥检查通过不可靠介质传输或存储的信息完整性的方法。 通常，信息验证码用于共享密钥的双方之间，以验证双方之间传输的信息。  
基于加密散列函数的 MAC 机制称为 HMAC。 HMAC 可与任何加密散列函数（如 MD5 或 SHA-1）和共享密钥结合使用。 HMAC 由 RFC 2104 规定。


## Cipher
**第一步**：`Cipher` 对象使用 `getInstance` 工厂方法创建：
```
public static Cipher getInstance(String transformation);
public static Cipher getInstance(String transformation,String provider);
```
`transformation` 是一个字符串，用于描述对给定输入执行的操作（或操作集），以产生某些输出。 转换总是包括加密算法的名称（如 AES/DES），后面还可以加上模式和填充方案，如下形式：
+ `algorithm/mode/padding`, 如 "DES/CBC/PKCS5Padding"
+ `algorithm` ， 如 "AES"

Algorithm:
+ `AES`: 支持 128, 192, 和 256 bits.
+ `Blowfish`: The block cipher designed by Bruce Schneier.
+ `DES`
+ `DESede`: Triple DES Encryption (DES-EDE).
+ `ElGamal`: Asymmetric key encryption based on Diffie–Hellman key exchange.
+ `MARS`: A shared-key (symmetric) block cipher, supporting 128-bit blocks and variable key size created by IBM.
+ `PBEWith<digest>And<encryption>`
+ `RC2` and `RC4`: Variable-key-size encryption algorithms developed by Ron Rivest for RSA Data Security, Inc.
+ `RSA`: The RSA encryption algorithm as defined in PKCS #1.
+ `Seal`: A software-efficient stream cipher by Phil Rogaway, IBM.

Mode:
+ `ECB`: Electronic Codebook Mode, as defined in: The National Institute of Standards and Technology (NIST) Federal Information Processing Standard (FIPS) PUB 81, DES Modes of Operation,U.S. + Department of Commerce, Dec 1980.
+ `CBC`: Cipher Block Chaining Mode, as defined in FIPS PUB 81.
+ `CFB`: Cipher Feedback Mode, as defined in FIPS PUB 81.
+ `CTR`: Counter Mode, as defined in NIST SP 800-38A.
+ `OFB`: Output Feedback Mode, as defined in FIPS PUB 81.
+ `PCBC`: Plaintext Cipher Block Chaining, as defined by Kerberos.
+ `CTS`: Cipher Text Stealing Mode, as defined in RFC 2040.

Padding:
+ `NoPadding`: No padding.
+ `PKCS5Padding`: The padding scheme described in: RSA Laboratories, PKCS #5: Password-Based Encryption Standard, version 1.5, November 1993.
+ `ISO10126Padding`: The padding scheme required by JSR 105 & JSR 106.

**第二步**：使用 `getInstance` 获取的密码对象必须针对四种模式之一进行初始化，这四种模式在密码类中定义为最终整数常量：
1. `Cipher.ENCRYPT_MODE`-加密模式
2. `Cipher.DECRYPT_MODE`-解密模式
3. `Cipher.WRAP_MODE`-将密钥封装成字节，以便安全地传输密钥。
4. `Cipher.UNWRAP_MODE`-将先前封装的密钥解包为 `java.security.Key` 对象。

每个密码初始化方法都需要一个模式参数（opmode），并根据该模式初始化密码对象。 其他参数还包括密钥（key）或包含密钥的证书（certificate）、算法参数（params）和随机性来源（random）：
```
public void init(int opmode, Key key);
public void init(int opmode, Certificate certificate)
public void init(int opmode, Key key, SecureRandom random);
public void init(int opmode, Certificate certificate, SecureRandom random)
public void init(int opmode, Key key,AlgorithmParameterSpec params);
public void init(int opmode, Key key,AlgorithmParameterSpec params,SecureRandom random);
public void init(int opmode, Key key,AlgorithmParameters params)
public void init(int opmode, Key key,AlgorithmParameters params,SecureRandom random)
```

**第三步**：加密或解密数据，数据加密或解密可以一步完成（单部分操作），也可以多步完成（多部分操作）。 如果事先不知道数据有多长，或者数据太长无法一次性存储在内存中，多部分操作就很有用。
单步操作：
```
public byte[] doFinal(byte[] input);
public byte[] doFinal(byte[] input, int inputOffset,int inputLen);
public int doFinal(byte[] input, int inputOffset, int inputLen, byte[] output);
public int doFinal(byte[] input, int inputOffset, int inputLen, byte[] output, int outputOffset)
```

多步操作：
```
public byte[] update(byte[] input);
public byte[] update(byte[] input, int inputOffset, int inputLen);
public int update(byte[] input, int inputOffset, int inputLen,byte[] output);
public int update(byte[] input, int inputOffset, int inputLen,byte[] output, int outputOffset)
```
多部分操作必须由前面的 `doFinal` 方法之一（如果最后一步仍有剩余输入数据）或下面的 `doFinal` 方法之一（如果最后一步没有剩余输入数据）终止：
```
public byte[] doFinal();
public int doFinal(byte[] output, int outputOffset);
```
`Cipher` 的某些 `update` 和 `doFinal` 方法允许调用者指定用于加密或解密数据的输出缓冲区。 在这些情况下，必须传递一个足够大的缓冲区，以容纳加密或解密操作的结果。
Cipher 中的以下方法可用于确定输出缓冲区的大小： `public int outOutputSize(int inputLen)` 。

调用 `doFinal` 会将 `Cipher` 对象重置为通过调用 `init` 初始化时的状态。 也就是说， `Cipher` 对象被重置后，可以继续加密或解密（取决于调用 `init` 时指定的操作模式）更多数据。

除了加密与解密之外， `Cipher` 还可以对密钥进行 `wrap/unwrap` 以实现密钥从一个地方到另一个地方的安全传输。由于 `wrap/unwrap` 应用程序接口可直接处理密钥对象，因此编写代码更加方便。 这些方法还可以安全地传输基于硬件的密钥。

要对密钥进行 `wrap` ，首先要初始化 `Cipher` 对象为 `WRAP_MODE` 模式，然后调用下面方法：
```
public final byte[] wrap(Key key);
```
如果您要将已封装的密钥字节（调用 `wrap` 的结果）提供给其他人，由其进行 `unwrap` ，请务必同时发送接收者进行解包所需的附加信息：
+ 密钥使用的算法 ， `Key` 接口的 `public String getAlgorithm();` 方法可以获取 `Key` 使用的算法
+ 密钥的类型(SECRET_KEY, PRIVATE_KEY,  PUBLIC_KEY)

有了以上信息，接受者首先要为 `UNWRAP_MODE` 初始化一个 `Cipher` 对象，然后调用下面的方法：
```
public final Key unwrap(byte[] wrappedKey,String wrappedKeyAlgorithm,int wrappedKeyType);
```
`wrappedKeyType` 是 `SECRET_KEY/PRIVATE_KEY/PUBLIC_KEY`中的一种。

## KeyGenerator/KeyPairGenerator
`KeyGenerator` 用于生成对称算法的密钥， `KeyPairGenerator` 用于生成公私钥对 。使用下述方法获取 `KeyGenerator` 对象：
```
public static KeyGenerator getInstance(String algorithm);
public static KeyGenerator getInstance(String algorithm,String provider);
```

`algorithm` 可以使用以下列表中的值：
+ `AES`
+ `Blowfish`
+ `DES`
+ `DESede`
+ `HmacMD2`
+ `HmacMD5`
+ `HmacSHA1`
+ `HmacSHA224`
+ `HmacSHA256`
+ `HmacSHA384`
+ `HmacSHA512`
+ `MARS`
+ `RC2`
+ `RC4`
+ `Seal`

生成密钥有两种方式：一种是独立于算法的方式，另一种是特定于算法的方式。 两者唯一的区别在于 `KeyGenerator` 的初始化方式。
1. Algorithm-Independent Initialization
```
public void init(SecureRandom random);
public void init(int keysize);
public void init(int keysize, SecureRandom random);
```

2. Algorithm-Specific Initialization
```
public void init(AlgorithmParameterSpec params);
public void init(AlgorithmParameterSpec params,SecureRandom random);
```

最后一步，生成 `Key` :
```
public SecretKey generateKey();
```

示例：
```java
KeyGenerator keyGenerator = KeyGenerator.getInstance(cipher);
keyGenerator.init(keySize);
Key key = keyGenerator.generateKey();

Cipher cipher = Cipher.getInstance("AES");
cipher.init(Cipher.ENCRYPT_MODE, key);
return new String(Base64.getEncoder().encode(cipher.doFinal(val.getBytes())));
```

## SecretKeyFactory
Key Factory 用于将密钥（java.security.Key 类型的不透明加密密钥）转换为密钥规范（以合适的格式对底层密钥材料进行透明表示，比如以证书表示），反之亦然（密钥<->证书或其他形式）。 `javax.crypto.SecretKeyFactory` 对象只处理秘密（对称）密钥，而 `java.security.KeyFactory` 对象则处理非对称算法密钥对中的公钥和私钥。

`java.security.Key`（其中 `java.security.PublicKey` 、`java.security.PrivateKey` 和 `javax.crypto.SecretKey` 都是子类）类型的对象是不透明密钥对象，因为你无法知道它们是如何实现的。 底层实现取决于提供商，可以是基于软件的，也可以是基于硬件的。 Key Factory 允许提供商提供自己的加密密钥实现。

例如，如果你有一个 Diffie Hellman 公钥的密钥规范，其中包括公开值 y、质数模 p 和基数 g，而你把相同的规范材料输入到不同提供商的 Diffie-Hellman 密钥工厂，那么得到的 PublicKey 对象很可能会有不同的底层实现。

```
byte[] desKeyData = { (byte)0x01, (byte)0x02, (byte)0x03, (byte)0x04, (byte)0x05, (byte)0x06, (byte)0x07, (byte)0x08 };
DESKeySpec desKeySpec = new DESKeySpec(desKeyData);
SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("DES");
SecretKey secretKey = keyFactory.generateSecret(desKeySpec);
```
在上面这种情况下， secretKey 的底层实现基于 keyFactory provider。 使用实现了 `javax.crypto.SecretKey` 接口的 `javax.crypto.spec.SecretKeySpec` 类，是利用相同的密钥材料创建功能等同的 `SecretKey` 对象的另一种独立于提供商的方法：
```
byte[] desKeyData = { (byte)0x01, (byte)0x02, ...};
SecretKeySpec secretKey = new SecretKeySpec(desKeyData, "DES");
```



## SealedObject
对于任何实现了 `java.io.Serializable` 接口的对象，您都可以创建一个 `SealedObject` ，以序列化格式（即 "深度拷贝"）封装原始对象，并使用 DES 等加密算法密封（加密）其序列化内容，以保护其机密性。 加密后的内容随后可以解密（使用正确的解密密钥通过相应的算法进行解密）和去序列化，从而得到原始对象。

```
Cipher c = Cipher.getInstance("DES");
c.init(Cipher.ENCRYPT_MODE, sKey);

// do the sealing
SealedObject so = new SealedObject("This is a secret", c);

c.init(Cipher.DECRYPT_MODE, sKey);
try {
	String s = (String)so.getObject(c);
} catch (Exception e) {
	// do something
};
```

## KeyAgreement
`KeyAgreement` 类提供了密钥协商协议的功能。 建立通信的双方进行秘密通信时需要密钥， 密钥通常情况下由密钥生成器（ `KeyPairGenerator` 或 `KeyGenerator` 、密钥工厂（ `KeyFactory` ）来生成。但是 `KeyAgreement` 可以允许通信双方在不交换密钥的情况下协商出一个密钥。

**第一步**，创建 `KeyAgreement` 对象：
```
public static KeyAgreement getInstance(String algorithm);
public static KeyAgreement getInstance(String algorithm,String provider);
```
algorithm 可以使用下述值：
+ `DiffieHellman`: Diffie-Hellman Key Agreement as defined in PKCS #3: Diffie-Hellman Key-Agreement Standard, RSA Laboratories, version 1.4, November 1993.
+ `ECDH`: Elliptic curve Diffie–Hellman.

**第二步**，初始化：
```
public void init(Key key);
public void init(Key key, SecureRandom random);
public void init(Key key, AlgorithmParameterSpec params);
public void init(Key key, AlgorithmParameterSpec params,SecureRandom random);
```

**第三步**，进行协商：
```
public Key doPhase(Key key, boolean lastPhase);
```
`key` 参数包含该阶段要处理的密钥。 在大多数情况下，这是参与密钥协议的其他各方的公钥，或者是前一阶段生成的中间密钥。 `doPhase` 可能会返回一个中间密钥，你可能需要把它发送给参与密钥协议的其他各方，以便他们在后续阶段进行处理。  
`lastPhase` 参数指定要执行的阶段是否是密钥协议的最后一个阶段： 如果值为 FALSE，则表示这不是密钥协议的最后一个阶段（后面还有更多阶段）；如果值为 TRUE，则表示这是密钥协议的最后一个阶段，密钥协议已完成，即可以调用 `generateSecret` 。

**最后一步**，生成共享密钥：
```
public byte[] generateSecret();
public int generateSecret(byte[] sharedSecret, int offset);
public SecretKey generateSecret(String algorithm);
```

示例：
```
import java.math.BigInteger;
import java.security.AlgorithmParameterGenerator;
import java.security.AlgorithmParameters;
import java.security.KeyFactory;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.PublicKey;
import java.security.spec.X509EncodedKeySpec;
import java.util.Arrays;

import javax.crypto.Cipher;
import javax.crypto.KeyAgreement;
import javax.crypto.SecretKey;
import javax.crypto.ShortBufferException;
import javax.crypto.interfaces.DHPublicKey;
import javax.crypto.spec.DHParameterSpec;

/**
 * This program executes the Diffie-Hellman key agreement protocol
 * between 2 parties: Alice and Bob.
 *
 * By default, preconfigured parameters (1024-bit prime modulus and base
 * generator used by SKIP) are used.
 * If this program is called with the "-gen" option, a new set of
 * parameters is created.
 */

public class DHKeyAgreement2 {

 private DHKeyAgreement2() {
 }

 public static void main(String argv[]) {
 try {
 String mode = "USE_SKIP_DH_PARAMS";

 DHKeyAgreement2 keyAgree = new DHKeyAgreement2();

 if (argv.length > 1) {
 keyAgree.usage();
 throw new Exception("Wrong number of command options");
 } else
 if (argv.length == 1) {
 if (!(argv[0].equals("-gen"))) {
 keyAgree.usage();
 throw new Exception("Unrecognized flag: " + argv[0]);
 }
 mode = "GENERATE_DH_PARAMS";
 }

 keyAgree.run(mode);
 } catch (Exception e) {
 System.err.println("Error: " + e);
 System.exit(1);
 }
 }

 private void run(String mode) throws Exception {

 DHParameterSpec dhSkipParamSpec;

 if (mode.equals("GENERATE_DH_PARAMS")) {
 // Some central authority creates new DH parameters
 System.out.println("Creating Diffie-Hellman parameters (takes VERY long) ...");
 AlgorithmParameterGenerator paramGen =
 AlgorithmParameterGenerator.getInstance("DH");
 paramGen.init(512);
 AlgorithmParameters params = paramGen.generateParameters();
 dhSkipParamSpec =
 (DHParameterSpec) params.getParameterSpec(DHParameterSpec.class);
 } else {
 // use some pre-generated, default DH parameters
 System.out.println("Using SKIP Diffie-Hellman parameters");
 dhSkipParamSpec = new DHParameterSpec(skip1024Modulus, skip1024Base);
 }

 /*
 * Alice creates her own DH key pair, using the DH parameters from
 * earlier code
 */
 System.out.println("ALICE: Generate DH keypair ...");
 KeyPairGenerator aliceKpairGen = KeyPairGenerator.getInstance("DH");
 aliceKpairGen.initialize(dhSkipParamSpec);
 KeyPair aliceKpair = aliceKpairGen.generateKeyPair();

 // Alice creates and initializes her DH KeyAgreement object
 System.out.println("ALICE: Initialization ...");
 KeyAgreement aliceKeyAgree = KeyAgreement.getInstance("DH");
 aliceKeyAgree.init(aliceKpair.getPrivate());

 // Alice encodes her public key, and sends it over to Bob.
 byte[] alicePubKeyEnc = aliceKpair.getPublic().getEncoded();

 /*
 * Let's turn over to Bob. Bob has received Alice's public key
 * in encoded format.
 * He instantiates a DH public key from the encoded key material.
 */
 KeyFactory bobKeyFac = KeyFactory.getInstance("DH");
 X509EncodedKeySpec x509KeySpec = new X509EncodedKeySpec(alicePubKeyEnc);
 PublicKey alicePubKey = bobKeyFac.generatePublic(x509KeySpec);

 /*
 * Bob gets the DH parameters associated with Alice's public key. 
 * He must use the same parameters when he generates his own key
 * pair.
 */
 DHParameterSpec dhParamSpec = ((DHPublicKey) alicePubKey).getParams();

 // Bob creates his own DH key pair
 System.out.println("BOB: Generate DH keypair ...");
 KeyPairGenerator bobKpairGen = KeyPairGenerator.getInstance("DH");
 bobKpairGen.initialize(dhParamSpec);
 KeyPair bobKpair = bobKpairGen.generateKeyPair();

 // Bob creates and initializes his DH KeyAgreement object
 System.out.println("BOB: Initialization ...");
 KeyAgreement bobKeyAgree = KeyAgreement.getInstance("DH");
 bobKeyAgree.init(bobKpair.getPrivate());

 // Bob encodes his public key, and sends it over to Alice.
 byte[] bobPubKeyEnc = bobKpair.getPublic().getEncoded();

 /*
 * Alice uses Bob's public key for the first (and only) phase
 * of her version of the DH
 * protocol.
 * Before she can do so, she has to instanticate a DH public key
 * from Bob's encoded key material.
 */
 KeyFactory aliceKeyFac = KeyFactory.getInstance("DH");
 x509KeySpec = new X509EncodedKeySpec(bobPubKeyEnc);
 PublicKey bobPubKey = aliceKeyFac.generatePublic(x509KeySpec);
 System.out.println("ALICE: Execute PHASE1 ...");
 aliceKeyAgree.doPhase(bobPubKey, true);

 /*
 * Bob uses Alice's public key for the first (and only) phase
 * of his version of the DH
 * protocol.
 */
 System.out.println("BOB: Execute PHASE1 ...");
 bobKeyAgree.doPhase(alicePubKey, true);

 /*
 * At this stage, both Alice and Bob have completed the DH key
 * agreement protocol.
 * Both generate the (same) shared secret.
 */
 byte[] aliceSharedSecret = aliceKeyAgree.generateSecret();
 int aliceLen = aliceSharedSecret.length;

 byte[] bobSharedSecret = new byte[aliceLen];
 int bobLen;
 try {
 // show example of what happens if you
 // provide an output buffer that is too short
 bobLen = bobKeyAgree.generateSecret(bobSharedSecret, 1);
 } catch (ShortBufferException e) {
 System.out.println(e.getMessage());
 }
 // provide output buffer of required size
 bobLen = bobKeyAgree.generateSecret(bobSharedSecret, 0);

 System.out.println("Alice secret: " + toHexString(aliceSharedSecret));
 System.out.println("Bob secret: " + toHexString(bobSharedSecret));

 if (!java.util.Arrays.equals(aliceSharedSecret, bobSharedSecret))
 throw new Exception("Shared secrets differ");
 System.out.println("Shared secrets are the same");

 /*
 * Now let's return the shared secret as a SecretKey object
 * and use it for encryption. First, we generate SecretKeys for the
 * "DES" algorithm (based on the raw shared secret data) and
 * then we use DES in ECB mode
 * as the encryption algorithm. DES in ECB mode does not require any
 * parameters.
 *
 * Then we use DES in CBC mode, which requires an initialization
 * vector (IV) parameter. In CBC mode, you need to initialize the
 * Cipher object with an IV, which can be supplied using the
 * javax.crypto.spec.IvParameterSpec class. Note that you have to use
 * the same IV for encryption and decryption: If you use a different
 * IV for decryption than you used for encryption, decryption will
 * fail.
 *
 * Note: If you do not specify an IV when you initialize the
 * Cipher object for encryption, the underlying implementation
 * will generate a random IV. You will have to retrieve this IV using the
 * javax.crypto.Cipher.getParameters() method, which returns an 
 * instance of java.security.AlgorithmParameters. You need to transfer
 * the contents of that object (for example, in encoded format, obtained 
 * using the AlgorithmParameters.getEncoded() method) to the party who will
 * do the decryption. When initializing the Cipher for decryption,
 * the (reinstantiated) AlgorithmParameters object must be passed to
 * the Cipher.init() method.
 */
 System.out.println("Return shared secret as SecretKey object ...");
 // Bob
 // Note: The call to bobKeyAgree.generateSecret reset the key
 // agreement object, so we call doPhase again prior to another
 // generateSecret call
 bobKeyAgree.doPhase(alicePubKey, true);
 SecretKey bobDesKey = bobKeyAgree.generateSecret("DES");

 // Alice
 // Note: The call to aliceKeyAgree.generateSecret reset the key
 // agreement object, so we call doPhase again prior to another
 // generateSecret call
 aliceKeyAgree.doPhase(bobPubKey, true);
 SecretKey aliceDesKey = aliceKeyAgree.generateSecret("DES");

 /*
 * Bob encrypts, using DES in ECB mode
 */
 Cipher bobCipher = Cipher.getInstance("DES/ECB/PKCS5Padding");
 bobCipher.init(Cipher.ENCRYPT_MODE, bobDesKey);

 byte[] cleartext = "This is just an example".getBytes();
 byte[] ciphertext = bobCipher.doFinal(cleartext);

 /*
 * Alice decrypts, using DES in ECB mode
 */
 Cipher aliceCipher = Cipher.getInstance("DES/ECB/PKCS5Padding");
 aliceCipher.init(Cipher.DECRYPT_MODE, aliceDesKey);
 byte[] recovered = aliceCipher.doFinal(ciphertext);

 if (!java.util.Arrays.equals(cleartext, recovered))
 throw new Exception(
 "DES in CBC mode recovered text is " + "different from cleartext");
 System.out.println("DES in ECB mode recovered text is " + "same as cleartext");

 /*
 * Bob encrypts, using DES in CBC mode
 */
 bobCipher = Cipher.getInstance("DES/CBC/PKCS5Padding");
 bobCipher.init(Cipher.ENCRYPT_MODE, bobDesKey);

 cleartext = "This is just an example".getBytes();
 ciphertext = bobCipher.doFinal(cleartext);
 // Retrieve the parameter that was used, and transfer it to Alice in
 // encoded format
 byte[] encodedParams = bobCipher.getParameters().getEncoded();

 /*
 * Alice decrypts, using DES in CBC mode
 */
 // Instantiate AlgorithmParameters object from parameter encoding
 // obtained from Bob
 AlgorithmParameters params = AlgorithmParameters.getInstance("DES");
 params.init(encodedParams);
 aliceCipher = Cipher.getInstance("DES/CBC/PKCS5Padding");
 aliceCipher.init(Cipher.DECRYPT_MODE, aliceDesKey, params);
 recovered = aliceCipher.doFinal(ciphertext);

 if (!java.util.Arrays.equals(cleartext, recovered))
 throw new Exception(
 "DES in CBC mode recovered text is " + "different from cleartext");
 System.out.println("DES in CBC mode recovered text is " + "same as cleartext");
 }

 /*
 * Converts a byte to hex digit and writes to the supplied buffer
 */
 private void byte2hex(byte b, StringBuffer buf) {
 char[] hexChars = { '0', '1', '2', '3', '4', '5', '6', '7', '8',
 '9', 'A', 'B', 'C', 'D', 'E', 'F' };
 int high = ((b & 0xf0) >> 4);
 int low = (b & 0x0f);
 buf.append(hexChars[high]);
 buf.append(hexChars[low]);
 }

 /*
 * Converts a byte array to hex string
 */
 private String toHexString(byte[] block) {
 StringBuffer buf = new StringBuffer();

 int len = block.length;

 for (int i = 0; i < len; i++) {
 byte2hex(block[i], buf);
 if (i < len - 1) {
 buf.append(":");
 }
 }
 return buf.toString();
 }

 /*
 * Prints the usage of this test.
 */
 private void usage() {
 System.err.print("DHKeyAgreement usage: ");
 System.err.println("[-gen]");
 }

 // The 1024 bit Diffie-Hellman modulus values used by SKIP
 private static final byte skip1024ModulusBytes[] = {
 (byte)0xF4, (byte)0x88, (byte)0xFD, (byte)0x58,
 (byte)0x4E, (byte)0x49, (byte)0xDB, (byte)0xCD,
 (byte)0x20, (byte)0xB4, (byte)0x9D, (byte)0xE4,
 (byte)0x91, (byte)0x07, (byte)0x36, (byte)0x6B,
 (byte)0x33, (byte)0x6C, (byte)0x38, (byte)0x0D,
 (byte)0x45, (byte)0x1D, (byte)0x0F, (byte)0x7C,
 (byte)0x88, (byte)0xB3, (byte)0x1C, (byte)0x7C,
 (byte)0x5B, (byte)0x2D, (byte)0x8E, (byte)0xF6,
 (byte)0xF3, (byte)0xC9, (byte)0x23, (byte)0xC0,
 (byte)0x43, (byte)0xF0, (byte)0xA5, (byte)0x5B,
 (byte)0x18, (byte)0x8D, (byte)0x8E, (byte)0xBB,
 (byte)0x55, (byte)0x8C, (byte)0xB8, (byte)0x5D,
 (byte)0x38, (byte)0xD3, (byte)0x34, (byte)0xFD,
 (byte)0x7C, (byte)0x17, (byte)0x57, (byte)0x43,
 (byte)0xA3, (byte)0x1D, (byte)0x18, (byte)0x6C,
 (byte)0xDE, (byte)0x33, (byte)0x21, (byte)0x2C,
 (byte)0xB5, (byte)0x2A, (byte)0xFF, (byte)0x3C,
 (byte)0xE1, (byte)0xB1, (byte)0x29, (byte)0x40,
 (byte)0x18, (byte)0x11, (byte)0x8D, (byte)0x7C,
 (byte)0x84, (byte)0xA7, (byte)0x0A, (byte)0x72,
 (byte)0xD6, (byte)0x86, (byte)0xC4, (byte)0x03,
 (byte)0x19, (byte)0xC8, (byte)0x07, (byte)0x29,
 (byte)0x7A, (byte)0xCA, (byte)0x95, (byte)0x0C,
 (byte)0xD9, (byte)0x96, (byte)0x9F, (byte)0xAB,
 (byte)0xD0, (byte)0x0A, (byte)0x50, (byte)0x9B,
 (byte)0x02, (byte)0x46, (byte)0xD3, (byte)0x08,
 (byte)0x3D, (byte)0x66, (byte)0xA4, (byte)0x5D,
 (byte)0x41, (byte)0x9F, (byte)0x9C, (byte)0x7C,
 (byte)0xBD, (byte)0x89, (byte)0x4B, (byte)0x22,
 (byte)0x19, (byte)0x26, (byte)0xBA, (byte)0xAB,
 (byte)0xA2, (byte)0x5E, (byte)0xC3, (byte)0x55,
 (byte)0xE9, (byte)0x2F, (byte)0x78, (byte)0xC7
 };


 // The SKIP 1024 bit modulus
 private static final BigInteger skip1024Modulus =
 new BigInteger(1, skip1024ModulusBytes);

 // The base used with the SKIP 1024 bit modulus
 private static final BigInteger skip1024Base = BigInteger.valueOf(2);
}
```


#### 非对称加密

非对称加密在需要一对公私钥对，所以在生成密钥时与对称加密算法有一点差别。

```java
Cipher rsaCipher = Cipher.getInstance("RSA");
KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
SecureRandom secureRandom = new SecureRandom();
keyPairGenerator.initialize(2048,secureRandom);
KeyPair keyPair = keyPairGenerator.generateKeyPair();
Key publicKey = keyPair.getPublic();
Key privateKey = keyPair.getPrivate();
rsaCipher.init(Cipher.UNWRAP_MODE,publicKey);
byte[] wrapKey = rsaCipher.wrap(aeskey);
```

### SSL证书主流的格式

仅在X.509证书中就有几种编码格式和扩展名。你可能见过.pem、.crt、.p7b、.der、pfx等等的证书。如果我们问是否所有的扩展名都是一样的，答案是既是又不是。所有这些证书都是一样的，但有不同的编码标准。所有这些都取决于你的应用程序，或操作系统接受的编码类型。

<center><img src="pics/x509.png" width="70%"></center>

SSL证书格式主要有公钥证书格式标准X.509中定义的PEM和DER、公钥密码标准PKCS中定义的PKCS#7和PKCS#12、Java环境专用的JKS。

PEM是基于Base64编码的证书格式，扩展名包括pem、crt和cer。Linux系统使用crt，Windows系统使用cer。PEM证书通常将根证书、中间证书和用户证书分开存放，主要用于Apache和Nginx。

DER是基于二进制编码的证书格式，扩展名包括der、crt和cer。DER证书相当于PEM证书的二进制版本，主要用于Java平台。

PKCS#7是基于Base64编码的证书格式，扩展名包括p7b和p7c。PKCS#7证书通常将根证书和中间证书合并存放，将用户证书单独存放。PKCS#7证书不包含私钥，主要用于Tomcat和Windows Server。

PKCS#12是基于二进制编码的证书格式，扩展名包括pfx和p12。PKCS#12证书通常将根证书、中间证书、用户证书和私钥合并存放并设置密码，主要用于Windows Server。

JKS是是基于二进制编码的证书格式，扩展名是jks。JKS证书通常将根证书、中间证书、用户证书和私钥合并存放并设置密码，主要用于Java Web Server。

证书颁发机构CA签发的证书通常是PEM格式或PKCS#7格式，而PKCS#12格式和JKS格式的证书需要进行证书格式转换才能得到。我们可以通过OpenSSL、Keytool或在线证书转换工具等方式将PEM格式或PKCS#7格式的证书转换为我们需要的其他格式。

#### SSL证书格式转换方法

常见SSL证书主要的文件类型和协议有: PEM、DER、PFX、JKS、KDB、CER、KEY、CSR、CRT、CRL、OCSP、SCEP等。而最常见的SSL证书格式之间的转换有：

1. 将JKS转换成PFX  
	可以使用Keytool工具，将JKS格式转换为PFX格式。
	```
		keytool -importkeystore -srckeystore D:\server.jks -destkeystore D:\server.pfx -srcstoretype JKS -deststoretype PKCS12
	```
2. 将PFX转换为JKS  
	可以使用Keytool工具，将PFX格式转换为JKS格式。
	```
	keytool -importkeystore -srckeystore D:\server.pfx -destkeystore D:\server.jks -srcstoretype PKCS12 -deststoretype JKS
	```
3. 将PEM/KEY/CRT转换为PFX  
	使用OpenSSL工具，可以将密钥文件KEY和公钥文件CRT转化为PFX文件。
	```
	openssl pkcs12 -export -out server.pfx -inkey server.key -in server.crt
	```
4. 将PFX转换为PEM/KEY/CRT  
	```
	openssl pkcs12 -in server.pfx -nodes -out server.pem
	openssl rsa -in server.pem -out server.key
	```