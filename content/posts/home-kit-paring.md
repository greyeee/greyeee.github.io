+++
title = 'Home Kit配对协议'
date = 2024-04-27T11:16:11+08:00
draft = false

+++

## SRP协议

SRP 是一种基于密码的相互身份验证的协议，要求双方都知道用户的密码。在身份验证过程中还生成了共享密钥，可通过对称加密来加密报文。

### 约定解释

```
N    A large safe prime (N = 2q+1, where q is prime)，All arithmetic is done modulo N.
g    A generator modulo N
k    Multiplier parameter (k = H(N, g) in SRP-6a, k = 3 for legacy SRP-6)
s    User's salt
I    Username
p    Cleartext Password
H()  One-way hash function
^    (Modular) Exponentiation
u    Random scrambling parameter
a,b  Secret ephemeral values
A,B  Public ephemeral values
x    Private key (derived from p and s)
v    Password verifier
```

```
N    安全的大素数，所有的运算都是mod N
g    A generator modulo N
k    Multiplier parameter (k = H(N, g) in SRP-6a, k = 3 for legacy SRP-6)
s    加盐，是一种随机产生的数据，用于保护密码的安全
I    用户名
p    明文密码
H()  单向哈希函数，不可逆
^    模幂运算，即求a^b mod n的值
u    随机混淆参数，用于增加安全性
a,b  秘密临时值，是随机生成的大整数
A,B  公开临时值，用于身份验证的过程中
x    私钥，由密码p和盐值s衍生而来
v    密码验证器，由g^x mod N计算得出，用于在服务器端验证用户的密码
```

### 生成和存储Password Verifier

通过以下公式生成password verifier，服务端将{I, s, v}存储到数据库，可通过索引键I获取该三元组。

```
s = randomsalt()          (same length as N)
I = H(I)
p = H(p)                  (hash/expand I & p)
t = H(I, ":", p)
x = H(s, t)
v = g^x                   (computes password verifier)
```

### 身份认证过程

```
Client                       Server
--------------               ----------------
un, pw = < user input >
I = H(un)
p = H(pw)
a = random()
A = g^a % N
            I, A -->
                          s, v = lookup(I)
                          b = random()
                          B = (kv + g^b) % N
                          u = H(A, B)
                          S = ((A * v^u) ^ b) % N
                          K = H(S)
                          M' = H(K, A, B, I, s, N, g)
             <-- s, B
u = H(A, B)
x = H(s, p)
S = ((B - k (g^x)) ^ (a + ux)) % N
K = H(S)
M = H(K, A, B, I, s, N, g)

            M -->
                          M must be equal to M'
                          Z = H(M, K)
            <-- Z

Z' = H(M, K)
Z' must equal Z
```

当服务器收到<I, A>，其中I是用户名，A是用户生成的公开临时值，服务器就可以生成共享密钥和生成证明M'。共享密钥K的生成需要password verifier。

生成证明：服务器和客户端都要计算一个值M，这个值是基于用户名、盐值、临时公开值和共享密钥等参数的哈希。服务器计算的值被称为M'。服务器通过验证M和M'是否相等，验证客户端是否生成和服务器一样的共享密钥，从而验证了客户端的身份，并生成证明Z给客户端，同理客户端通过验证Z和Z'是否相等，验证了服务器身份。

```go
package main

import (
	"crypto/subtle"
	"fmt"
	"github.com/opencoff/go-srp"
)

func main() {
	bits := 1024
	pass := []byte("password string that's too long")
	i := []byte("foouser")

	s, err := srp.New(bits)
	if err != nil {
		panic(err)
	}

	v, err := s.Verifier(i, pass)
	if err != nil {
		panic(err)
	}

	ih, vh := v.Encode()

	// Store ih, vh in durable storage
	fmt.Printf("Verifier Store:\n   %s => %s\n", ih, vh)

	c, err := s.NewClient(i, pass)
	if err != nil {
		panic(err)
	}

	// client credentials (public key and identity) to send to server
	creds := c.Credentials()

	fmt.Printf("Client Begin; <I, A> --> server:\n   %s\n", creds)

	// Begin the server by parsing the client public key and identity.
	ih, A, err := srp.ServerBegin(creds)
	if err != nil {
		panic(err)
	}

	// Now, pretend to lookup the user db using "I" as the key and
	// fetch salt, verifier etc.
	s, v, err = srp.MakeSRPVerifier(vh)
	if err != nil {
		panic(err)
	}

	fmt.Printf("Server Begin; <v, A>:\n   %s\n   %x\n", vh, A.Bytes())
	srv, err := s.NewServer(v, A)
	if err != nil {
		panic(err)
	}

	// Generate the credentials to send to client
	creds = srv.Credentials()

	// Send the server public key and salt to server
	fmt.Printf("Server Begin; <s, B> --> client:\n   %s\n", creds)

	// client processes the server creds and generates
	// a mutual authenticator; the authenticator is sent
	// to the server as proof that the client derived its keys.
	cauth, err := c.Generate(creds)
	if err != nil {
		panic(err)
	}

	fmt.Printf("Client Authenticator: M --> Server\n   %s\n", cauth)

	// Receive the proof of authentication from client
	proof, ok := srv.ClientOk(cauth)
	if !ok {
		panic("client auth failed")
	}

	// Send proof to the client
	fmt.Printf("Server Authenticator: Z --> Client\n   %s\n", proof)

	// Verify the server's proof
	if !c.ServerOk(proof) {
		panic("server auth failed")
	}

	// Now, we have successfully authenticated the client to the
	// server and vice versa.

	kc := c.RawKey()
	ks := srv.RawKey()

	if 1 != subtle.ConstantTimeCompare(kc, ks) {
		panic("Keys are different!")
	}

	fmt.Printf("Client Key: %x\nServer Key: %x\n", kc, ks)
}
```

a

```
Verifier Store:
   b8fd39a36525c3a6aa40660f6093b2ae5024d23dcd1d9da9421ad53989fb07f8 => 128:eeaf0ab9adb38dd69c33f80afa8fc5e86072618775ff3c0b9ea2314c9c256576d674df7496ea81d3383b4813d692c6e0e0d5d8e250b98be48e495c1d6089dad15dc7d7b46154d6b6ce8ef4ad69b15d4982559b297bcf1885c529f566660e57ec68edbc3c05726cc02fd4cbf4976eaa9afd5138fe8376435b9fc61d2fc0eb06e3:2:17:b8fd39a36525c3a6aa40660f6093b2ae5024d23dcd1d9da9421ad53989fb07f8:19a5dd80011dc63a3dfda1c742d55399c009617987868a6951ea91d754d7d9519a3c4801ec0da71eb4ae1cf3f0f97c857954a4d8f54729170512312819c790296b12c84c28bbe270f542c89c6446591373d3fea8ff445a51cefe85d002ce32e1fb003c5533d6b97f1464680c88afd4cc9e15ce15163a410be44b1af9c5e08265:a7d956665ae1415b380db321bcca75f013bda5ea347489318d2c967d1b69e4115c54022225e67e5008fdfcf0841feeb5a0e26fabdb6f0e05b3fc9f96f6b830ab1599ae1b22cbc31cd9b667b48ef0a33d580e43999b2299c8acb0ec761cb72558cf2cf8a7bad78a5b5bbb8a2c473d2f93e13975618b54a431a38344d779ddb7e6
Client Begin; <I, A> --> server:
   b8fd39a36525c3a6aa40660f6093b2ae5024d23dcd1d9da9421ad53989fb07f8:1588ca95bcc97f38c88f5cc942712e27867e71f7e33c25d53e17142170152d0f03420a3d17f23e9451c335b713bb5143bc7c8f398239a923dacf5d186dd49902e01e6bd7fb14c46d0d41d0e328711e3182b2d86ccd9151db6555bbdd9aa5f5e7df655d3c910c1e2d3527463b4b5932693291eae189446416534b1afe77773916
Server Begin; <v, A>:
   128:eeaf0ab9adb38dd69c33f80afa8fc5e86072618775ff3c0b9ea2314c9c256576d674df7496ea81d3383b4813d692c6e0e0d5d8e250b98be48e495c1d6089dad15dc7d7b46154d6b6ce8ef4ad69b15d4982559b297bcf1885c529f566660e57ec68edbc3c05726cc02fd4cbf4976eaa9afd5138fe8376435b9fc61d2fc0eb06e3:2:17:b8fd39a36525c3a6aa40660f6093b2ae5024d23dcd1d9da9421ad53989fb07f8:19a5dd80011dc63a3dfda1c742d55399c009617987868a6951ea91d754d7d9519a3c4801ec0da71eb4ae1cf3f0f97c857954a4d8f54729170512312819c790296b12c84c28bbe270f542c89c6446591373d3fea8ff445a51cefe85d002ce32e1fb003c5533d6b97f1464680c88afd4cc9e15ce15163a410be44b1af9c5e08265:a7d956665ae1415b380db321bcca75f013bda5ea347489318d2c967d1b69e4115c54022225e67e5008fdfcf0841feeb5a0e26fabdb6f0e05b3fc9f96f6b830ab1599ae1b22cbc31cd9b667b48ef0a33d580e43999b2299c8acb0ec761cb72558cf2cf8a7bad78a5b5bbb8a2c473d2f93e13975618b54a431a38344d779ddb7e6
   1588ca95bcc97f38c88f5cc942712e27867e71f7e33c25d53e17142170152d0f03420a3d17f23e9451c335b713bb5143bc7c8f398239a923dacf5d186dd49902e01e6bd7fb14c46d0d41d0e328711e3182b2d86ccd9151db6555bbdd9aa5f5e7df655d3c910c1e2d3527463b4b5932693291eae189446416534b1afe77773916
Server Begin; <s, B> --> client:
   19a5dd80011dc63a3dfda1c742d55399c009617987868a6951ea91d754d7d9519a3c4801ec0da71eb4ae1cf3f0f97c857954a4d8f54729170512312819c790296b12c84c28bbe270f542c89c6446591373d3fea8ff445a51cefe85d002ce32e1fb003c5533d6b97f1464680c88afd4cc9e15ce15163a410be44b1af9c5e08265:c19dda117838c7635838cfb9fe6a989f7b897a28b37a931fb1705a0e2dc2c81f2b6848a0e729522328cb273ed1abc676f4edb46a113ea9f8933088ffa59682115c4a945ee199516f5f239bfb7df2bcde6b9d704079cd5144bc4adfbaec472995c36f93cdebf24b7e3dc0cfcb21888921e0582fec28da77e07c0dfc7b7ebb57ad
Client Authenticator: M --> Server
   201592da93ce191af56559836f9fa3960016f0b137b2bde4164692febb9fcdaf
Server Authenticator: Z --> Client
   cdb200ca5280baedff73efb4b4078b411bd6caf4b0456793b2a489ee16cbc2aa
Client Key: a13788fbd82def1995bf1057b32d6ea2c31a25ce056c1edb3bf766ce18ac09d6
Server Key: a13788fbd82def1995bf1057b32d6ea2c31a25ce056c1edb3bf766ce18ac09d6
```

![https://docs.espressif.com/projects/esp-idf/zh_CN/v5.1.2/esp32/_images/seqdiag-5054c1e6eebf7b8d04573fcd01575dcd936db1b5.png](https://docs.espressif.com/projects/esp-idf/zh_CN/v5.1.2/esp32/_images/seqdiag-5054c1e6eebf7b8d04573fcd01575dcd936db1b5.png)

## Home Kit配对

home kit配对使用SRP协议，username和password为“Pair‐Setup”和setup code。

### 配对流程

```
iOS device/srp client                                                Accessory/srp server
--------------                                                       ----------------
           			  SRP Start Request ->
           			  kTLVType_State <M1>
           			  kTLVType_Method <Pair Setup with Authentication>
                                                                     SRP6a_server_method()
                                                                     SRP_set_username() with "Pair‐Setup"
                                                                     retrieve SRP verifier for the setup code
                                                                     from an EEPROM and SRP_set_authenticator()
                                                                     salt = random()
                                                                     public key = SRP_gen_pub()
           			  <-- SRP Start Response
           			  kTLVType_State <M2>
           			  kTLVType_PublicKey <Accessory’s SRP public key>
           			  kTLVType_Salt <16 byte salt>
                  
SRP6a_client_method()
SRP_set_username() with "Pair‐Setup"
SRP_set_params() with salt
public key = SRP_gen_pub()
shared secret key = SRP_compute_key()
proof = SRP_respond()

           			  --> SRP Verify Request
           			  kTLVType_State <M3>
           			  kTLVType_PublicKey <iOS device’s SRP public key>
           			  kTLVType_Proof <iOS device’s SRP proof>
                                                                    shared secret key = SRP_compute_key() using
                                                                    public key
                                                                    verify proof with SRP_verify()
                                                                    proof = SRP_respond()
                                                                    
                                                                    MFiProof和Accessory Certificate
                                                                    derive MFiChallenge from shared secret by 
                                                                    using HKDF‐SHA‐512.
                                                                    MFiChallengeData = SHA‐256(MFiChallenge)
                                                                    Generate the MFi proof, MFiChallengeData 
                                                                    written to the ”Challenge Data” register of 
                                                                    the Apple Authentication Coprocessor.
                                                                    Read Accessory Certificate from Apple 
                                                                    Authentication Coprocessor.
                                                                    Derive SessionKey from shared secret by 
                                                                    using HKDF‐SHA‐512.
                                                                    sub‐TLV
                                                                    kTLVType_Signature <MFiProof> 
                                                                    kTLVType_Certificate <AccessoryCertificate>
                                                                    encryptedData, authTag = 
                                                                    ChaCha20‐Poly1305(SessionKey, 
                                                                    Nonce=”PS‐Msg04”, AAD=<none>, Msg=<Sub‐TLV>)
           			  <-- SRP Verify Response                    
           			  kTLVType_State <M4>
           			  kTLVType_Proof <Accessory’s SRP proof>
           			  kTLVType_EncryptedData <encryptedData with authTag appended>
                  
Verify proof with SRP_verify()
Derive SessionKey from shared secret by using HKDF‐SHA‐512.
Verify sub‐TLV’s authTag from encrypted Data.
Decrypt sub‐TLV in encryptedData
Verify AccessoryCertificate, must be issued by the MFi Sub‐Certificate Authority, chained to the Apple root certificate, and have a class between 0x00 and 0x0A.
Derive MFiChallenge from shared secret by using HKDF‐SHA‐512.
Verify MFiProof using the public key contained in the Accessory Certificate.

Generate Ed25519 long‐term public key, iOSDeviceLTPK, and long‐term secret key, iOSDeviceLTSK.
Derive iOSDeviceX from shared secret by using HKDF‐SHA‐512.
Concatenate iOSDeviceX with the iOS device’s Pairing Identifier, iOSDevicePairingID,and its long‐term public key, iOSDeviceLTPK. reffered to as iOSDeviceInfo.
Generate iOSDeviceSignature by signing iOSDeviceInfo with its long‐term secret key, iOSDeviceLTSK, using Ed25519.
sub‐TLV
kTLVType_Identifier <iOSDevicePairingID>
kTLVType_PublicKey <iOSDeviceLTPK>
kTLVType_Signature <iOSDeviceSignature>
Generate the 16 byte authtag.
encryptedData, authTag = ChaCha20‐Poly1305(SessionKey, Nonce=”PS‐Msg05”, AAD=<none>, Msg=<Sub‐TLV>)

           			  --> Exchange Request
           			  kTLVType_State <M5>
           			  kTLVType_EncryptedData <encryptedData with authTag appended>
                                                                    Verify sub‐TLV’s authTag from encrypted 
                                                                    Data.
                                                                    Decrypt sub‐TLV in encryptedData.
                                                                    Derive iOSDeviceX from shared secret by 
                                                                    using HKDF‐SHA‐512.
                                                                    Concatenate iOSDeviceX with the iOS device’s 
                                                                    Pairing Identifier, iOSDevicePairingID, and 
                                                                    its long‐term public key, iOSDeviceLTPK. 
                                                                    reffered to as iOSDeviceInfo.
                                                                    Use Ed25519 to verify the signature of the 
                                                                    constructed iOSDeviceInfo with the
                                                                    iOSDeviceLTPK from the decrypted sub‐TLV.
                                                                    Persistently save the iOSDevicePairingID and 
                                                                    iOSDeviceLTPK as a pairing.
                                                                    
                                                                    Generate its Ed25519 long‐term public key, 
                                                                    AccessoryLTPK, and long‐term secret key, 
                                                                    AccessoryLTSK.
                                                                    Derive AccessoryX from  shared secret by 
                                                                    using HKDF‐SHA‐512.
                                                                    Concatenate AccessoryX with the accessory’s 
                                                                    PairingIdentifier, AccessoryPairingID, and 
                                                                    its long‐term public key, AccessoryLTPK. 
                                                                    Use Ed25519 to generate AccessorySignature 
                                                                    by signing AccessoryInfo with its long‐term 
                                                                    secret key, AccessoryLTSK.
                                                                    sub‐TLV
                                                                    kTLVType_Identifier <AccessoryPairingID>
                                                                    kTLVType_PublicKey <AccessoryLTPK>
                                                                    kTLVType_Signature <AccessorySignature>
                                                                    Generate the 16 byte authtag.
                                                                    encryptedData, authTag = 
                                                                    ChaCha20‐Poly1305(SessionKey, 
                                                                    Nonce=”PS‐Msg06”, AAD=<none>, Msg=<Sub‐TLV>)
           			  <-- Exchange Response          
           			  kTLVType_State <M6>
           			  kTLVType_EncryptedData <encryptedData with authTag appended>
Verify sub‐TLV’s authTag from encrypted Data.
Decrypt sub‐TLV in encryptedData.
Uses Ed25519 to verify the signature of AccessoryInfo using AccessoryLTPK.
Persistently saves AccessoryPairingID and AccessoryLTPK as a pairing.
```

### iOS device验证Accessory是MFI认证的

Accessory

1. 从SRP共享密钥派生MFiChallenge，传给MFI芯片生成MFi proof，从MFI芯片读取Accessory Certificate。
2. 从SRP共享密钥派生SessionKey，用ChaCha20‐Poly1305加密算法使用SessionKey加密MFi proof和Accessory Certificate。

iOS device

1. 从SRP共享密钥派生SessionKey，用ChaCha20‐Poly1305加密算法使用SessionKey解密得到MFi proof和Accessory Certificate。
2. 验证Accessory Certificate。
3. 从SRP共享密钥派生MFiChallenge，用Accessory Certificate中的public key验证MFi proof。

### 为什么从SRP共享密钥派生SessionKey，而不直接用SRP共享密钥

派生出SessionKey来进行部分操作，可以增加安全性。即使SessionKey被泄露，攻击者也无法利用这个密钥进行其他操作，因为它只用于特定的步骤。而且，每个SessionKey都是临时的，只用于一次会话，会话结束后就会废弃，这样也能防止长时间使用同一密钥导致的安全风险。

### SRP身份认证和MFI验证结束后，双方交换对方的PairingID、公钥保存下来

加密是为了保护数据的隐私，签名是为了验证数据的完整性和身份验证，防止数据被篡改。

iOS device

1. Ed25519生成公钥、私钥，从SRP共享密钥派生iOSDeviceX。
2. 用私钥对<iOSDeviceX + iOSDevicePairingID + 公钥>生成签名。
3. 用ChaCha20‐Poly1305加密算法使用SessionKey加密iOSDevicePairingID、iOSDeviceLTPK、iOSDeviceSignature。

Accessory

1. 用ChaCha20‐Poly1305加密算法使用SessionKey解密得到iOSDevicePairingID、iOSDeviceLTPK、iOSDeviceSignature。
2. 从SRP共享密钥派生iOSDeviceX，用Ed25519算法验证<iOSDeviceX + iOSDevicePairingID + 公钥>的签名，使用公钥iOSDeviceLTPK。
3. Ed25519生成公钥、私钥，从SRP共享密钥派生AccessoryX。
4. 用私钥对<AccessoryX + AccessoryPairingID + 公钥>生成签名。
5. 用ChaCha20‐Poly1305加密算法使用SessionKey加密AccessoryPairingID、AccessoryLTPK、AccessorySignature。

iOS device

1. 用ChaCha20‐Poly1305加密算法使用SessionKey解密得到AccessoryPairingID、AccessoryLTPK、AccessorySignature。
2. 从SRP共享密钥派生AccessoryX，用Ed25519算法验证<AccessoryX + AccessoryPairingID + 公钥>的签名，使用公钥AccessoryLTPK。

参考：

https://en.wikipedia.org/wiki/Secure_Remote_Password_protocol

https://github.com/opencoff/go-srp

http://srp.stanford.edu/ndss.html

https://pythonhosted.org/srp/srp.html

https://docs.espressif.com/projects/esp-idf/zh_CN/v5.1.2/esp32/api-reference/provisioning/provisioning.html
