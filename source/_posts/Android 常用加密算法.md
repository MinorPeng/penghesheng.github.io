---
title: Android 常用加密算法
date: 2020-03-02
tag: Android
category: Android

---

[toc]

# Android 常用加密算法

**数字签名**：

简单来说就是提供可鉴别的数字信息验证自身身份的一种方式。一套数字签名通常定义两种互补的运算，一个用于签名，另一个用于验证。分别又发送者持有能够代表自己身份的私钥（不可泄漏），由接受者持有与私钥对应的公钥，能够在接受到来自发送者信息时用于验证身份。

## 对称加密

加密和解密密钥相同，加解密过程可逆

- DES

    Data Encryption Standard，数据加密算法

    > Key：8字节
    >
    > Data：8字节，数据
    >
    > Mode：DES工作模式，有EBC（电子密码本模式）、CBC（加密分组链接模式  ）、CFB（加密反馈模式）、OFB（输出反馈模式）、CTR（计数器模式）；NoPadding（不填充）、Zeros填充（0填充）、PKCS5Padding填充

    示例：

    ```kotlin
    @SuppressLint("GetInstance")
    fun desEncrypt(content: ByteArray, pwd: ByteArray): ByteArray {
        val random = SecureRandom()
        val desKey = DESKeySpec(pwd)
        val keyFactory = SecretKeyFactory.getInstance("DES")
        val secureKey = keyFactory.generateSecret(desKey)
        val cipher = Cipher.getInstance("DES")
        cipher.init(Cipher.ENCRYPT_MODE, secureKey, random)
        return cipher.doFinal(content)
    }
    
    @SuppressLint("GetInstance")
    fun desDecrypt(content: ByteArray, pwd: ByteArray): ByteArray {
        val random = SecureRandom()
        val desKey = DESKeySpec(pwd)
        val keyFactory = SecretKeyFactory.getInstance("DES")
        val secureKey = keyFactory.generateSecret(desKey)
        val cipher = Cipher.getInstance("DES")
        cipher.init(Cipher.DECRYPT_MODE, secureKey, random)
        return cipher.doFinal(content)
    }
    ```

    

- 3DES

    Triple DES，三重数据加密算法，相当于是对每个数据块应用三次数据加密标准（DES）算法，加长了密钥长度

    ```kotlin
    fun tripleDesEncrypt(data: ByteArray, key: ByteArray): ByteArray {
        val desKey = DESedeKeySpec(key)
        val keyFactory = SecretKeyFactory.getInstance("desede")
        val secureKey = keyFactory.generateSecret(desKey)
        val cipher = Cipher.getInstance("desede/ECB/PKCS5Padding")
        cipher.init(Cipher.ENCRYPT_MODE, secureKey)
        return cipher.doFinal(data)
    }
    
    fun tripleDesDecrypt(data: ByteArray, key: ByteArray): ByteArray {
        val desKey = DESedeKeySpec(key)
        val keyFactory = SecretKeyFactory.getInstance("desede")
        val secureKey = keyFactory.generateSecret(desKey)
        val cipher = Cipher.getInstance("desede/ECB/PKCS5Padding")
        cipher.init(Cipher.DECRYPT_MODE, secureKey)
        return cipher.doFinal(data)
    }
    ```

    

- AES

    Advanced Encryption Standard，高级加密标准，取代DES而出现

    ```kotlin
    fun aesEncrypt(data: ByteArray, key: ByteArray): ByteArray {
        val keygen = KeyGenerator.getInstance("AES")
        keygen.init(128, SecureRandom(key))
        val cipher = Cipher.getInstance("AES")
        cipher.init(Cipher.ENCRYPT_MODE, SecretKeySpec(keygen.generateKey().encoded, "AES"))
        return cipher.doFinal(data)
    }
    
    fun aesDecrypt(data: ByteArray, key: ByteArray): ByteArray {
        val keygen = KeyGenerator.getInstance("AES")
        keygen.init(128, SecureRandom(key))
        val cipher = Cipher.getInstance("AES")
        cipher.init(Cipher.DECRYPT_MODE, SecretKeySpec(keygen.generateKey().encoded, "AES"))
        return cipher.doFinal(data)
    }
    ```

    

## 非对称加密

加密和解密密钥不同

- RSA

    目前最有影响力的公钥加密算法，能抵抗目前已知所有密码攻击

    公钥私钥对

- DSA

    Schnorr和ElGamal签名算法的变种，被美国NIST作为DSS(DigitalSignature  Standard)。它是一种公开密钥算法，用作数字签名。DSA加密算法使用公开密钥，为接受者验证数据的完整性和数据发送者的身份，它也可用于由第三方去确定签名和所签数据的真实性

- ECC

    它比其他的方法使用 **更小的密钥**，比如 `RSA` **加密算法**，提供 **相当的或更高等级** 的安全级别。不过一个缺点是 **加密和解密操作** 的实现比其他机制 **时间长** (相比 `RSA` 算法，该算法对 `CPU` 消耗严重)

## 其他散列算法

摘要算法，不可逆过程，经过算法运算后都是生成固定长度的数据，使用16进行进行显示

- SHA-1

    160位摘要，强度更高

    ```kotlin
    val shaDigest = MessageDigest.getInstance("SHA-1")
    shaDigest.update(bytes)
    return shaDigest.digest()
    ```

    

- MD5

    1228位摘要，速度更快

    ```kotlin
    val shaDigest = MessageDigest.getInstance("MD5")
    shaDigest.update(bytes)
    return shaDigest.digest()
    ```

    

其他摘要算法类似

## 对比

![算法比较](https://raw.githubusercontent.com/MinorPeng/Image/master/android_encryption_compare.png)

## 参考

- [浅谈常见的七种加密算法及实现](https://juejin.im/post/5b48b0d7e51d4519962ea383)
- [Java DES 加密 解密 示例](https://blog.csdn.net/Techzero/article/details/17282637)
- [Java实现DES加密解密算法](https://blog.csdn.net/mrli113/article/details/72884301)
- [分组密码工作模式](https://zh.wikipedia.org/wiki/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)
- [常见对称加密算法与工作模式简介](https://blog.csdn.net/alwaysrun/article/details/89076403)
- [JAVA AES加密与解密](https://blog.csdn.net/u011781521/article/details/77932321)
- [常用加密解密算法【RSA、AES、DES、MD5】介绍和使用](https://blog.csdn.net/u013565368/article/details/53081195)
- [DSA加密算法以及破解](https://blog.csdn.net/happen_if/article/details/85219306)

