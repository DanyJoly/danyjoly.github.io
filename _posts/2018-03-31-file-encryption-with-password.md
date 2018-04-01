---
layout: post
title: File Encryption with Password
comments: true
---

About a month ago I wrote a post about cryptography tools. In this article, I use some of these tools to create a convenient data encryption protocol for small files.

# Security Disclaimer
Unless you're an expert (I'm not), using existing software that stood the test of time such as [VeraCrypt](https://www.veracrypt.fr/) or [OpenPGP](https://www.openpgp.org/) is better than rolling your own solution. Even gluing existing and proven cryptography libraries require non-trivial expertise. The code provided here is to learn about cryptography, but unless multiple world-class experts give me their thumb up (could take a while), don't rely on it being flawless.

# Requirements
I had a couple of ideas in mind that require strong file encryption. One of them is a simple [command line tool](https://github.com/DanyJoly/go-filecrypt) to encrypt files.

I wanted an encryption protocol that would:
* only require the user to provide a password. No passing around asymmetric key like OpenPGP, etc.
* be better than average against brute force attack. Because I plan to use this protocol on the client side, I don't need to think about server performance so encryption strength is all that matters.  
* detect malicious data tampering and raise an error if it's detected.

# Inspiration
I took inspiration from the VeraCrypt implementation. VeraCrypt is a handy tool that can mount encrypted files as drive partitions. It's itself a continuation of the now defunct TrueCrypt encryption tool, which has been extensively [audited by security experts](https://www.schneier.com/blog/archives/2015/04/truecrypt_secur.html). That way, I hope to alleviate many of the potential security issues stemming from improper architecture that could arise during this little educational project.

# The Solution
The typical solution for the encryption of a large quantity of data is with a symmetric key. It's fast, and implementations like AES are very robust if appropriately used.

Just like VeraCrypt and BitLocker (Microsoft), I decided to use AES-256 in XTS mode.  It's a modern block cipher developed for disk encryption. Other more traditional modes like CBC encryption would have been fine too, but XTS is a bit [less malleable](https://crypto.stackexchange.com/questions/5587/what-is-the-advantage-of-xts-over-cbc-mode-with-diffuser/5593) (tamper-resistant).

Unfortunately, while AES provides fast content encryption, it's not a complete solution. Two problems remain:
* AES keys are not the same thing as a password. AES keys are fixed-length, long (I use 256 bits), and unlike user passwords, they must have excellent entropy. We can't rely on the user to provide a password that respects these requirements.
* As mentioned previously, XTS is *less* malleable, not impervious to it. We need something to protect ourselves against data tampering.

## Password to AES key
To create fixed-length AES keys with excellent entropy, we rely on password hash functions. They have been developed precisely for that purpose. By being voluntarily expensive to compute, brute force attempt to get to the AES key is infeasible. They are also designed to spread the entropy to the full length of the key.

## Protection Against Data Tampering

As mentioned in my [previous post](http://localhost:4000/2018/02/26/cryptography-building-blocks-overview.html), the most common way to detect data tampering is to calculate a cryptographic hash of the content. Obviously, you then need to keep the hash securely around until the content is decrypted and compare the hash of the decrypted data to the original hash. This is called a message authentication code (MAC).

# The Implementation

Our first step is to generate an AES key from the user-provided password.

For this solution, I decided to use the Argon2, a new generation password hash function with the goal of adding additional protection against the latest GPU-based brute force attacks. Argon2 does this by also requiring a large amount of memory to compute the hash, something for which GPUs are not optimized.

Because Argon2 was released very recently (2015), and the jury is still out on its long-term robustness, I didn't feel safe relying solely on it. For that reason, I'm also applying a PBKDF2 password hash function serially with Argon2, using the same settings as VeraCrypt, which are already much more stringent than what the NIST recommended as of June 2017.

The first step is to generate a salt for the password hash function:  
```go
func GenerateSalt() (Salt, error) {
  b := make([]byte, saltLen)
  l, e := rand.Read(b) // crypto.rand
  if e != nil {
    return nil, e
  }
  if l != saltLen {
    return nil, fmt.Errorf("wrong salt length generated. Expected %d, got %d", saltLen, l)
  }

  return Salt(b), nil
}
```
The important part here is to use a cryptographic pseudo-random function. Regular pseudo-random functions are too predictable to be used in cryptography.

We can then use the salt to generate the AES-key.

```go
// AES-256 key used in XTS mode.
// 256 bits is a requirement of AES-256. To use XTS mode, we double it From the
// XTS documentation: "The key must be twice the length of the underlying
// cipher's key."
const aesKeyLen = 32 * 2

// VeraCrypt uses 50,000 itr. NIST recommends at least 10,000 itr.
const pbkdf2ItrCount = 50000
const pbkdf2KeyLen = aesKeyLen * 4
var pbkdf2Hash = sha512.New // SHA-512 matches VeraCrypt.

const argon2Time = 20          // Recommended for 64MB memory: at least 1
const argon2Memory = 64 * 1024 // size of the memory in KiB (64MB)
const argon2Threads = 4

// (...)

// aesKeyFromPasswordAndSalt will create the AES key using password hashing.
func aesKeyFromPasswordAndSalt(password []byte, salt []byte) ([]byte, error) {

  // We apply PBKDF2 and Argon2i one after the other. Same salt for both.
  pbkdf2Key := pbkdf2.Key(
          password, salt, pbkdf2ItrCount, pbkdf2KeyLen, pbkdf2Hash)
  aesKey := argon2.IDKey(
          pbkdf2Key, salt, argon2Time, argon2Memory, argon2Threads, aesKeyLen)

  return aesKey, nil
}
```
Luckily, The Go project has implemented both Argon2 and PBKDF2, so there is no need to link to the C implementations or to rely on less trusted 3rd party implementations.

You'll also notice that we generate two keys. (see aesKeyLen). AES XTS, unlike other AES modes, requires two keys.

We can then generate a XTS encryption cipher:

```go
cipher, e := xts.NewCipher(aes.NewCipher, aesKey)
if e != nil {
  return nil, e
}
```

Before encrypting the data though, I calculate the security hash on the plaintext:

```go
// Create a SHA-512 digest on the plain content to detect data tampering
digest := sha512.Sum512(plaintext)
```

## Encryption Protocol
All the previous steps were relatively straightforward and right out of the book. Now come the interesting choices.

We now know that we need to keep around multiple pieces of information so that we can decrypt the data later: the encrypted data itself, the salt used to hash the password and the digest used as a MAC.

We actually have a few more. Here is the final data format I ended up with for this implementation:
```
Data Format

  (64 bytes) salt
  (? bytes)  Encrypted content

  Encrypted content format:
  (6 bytes)  Magic bytes ('GOODPW')
  (1 byte)   Protocol version
  (1 byte)   Padding length (0 to 16 bytes)
  (? bytes)  Padding
  (64 bytes) MAC Digest
  (? bytes)  plain content
```

Let's take those steps by steps:

## Salt
That's the salt in plaintext used to encrypt the data. Because the password is secret, the salt can be public.

## Magic bytes
That's an idea I got from VeraCrypt/TrueCrypt. Once you decrypt the data, it's hard to tell if the password was correct or not. Notably, it's hard to distinguish if the password is wrong or if the data has been tampered with. If the magic bytes fit the expected value (GOODPW), it's *probably* the right password. I say "probably" because even though multiple keys could decrypt the same values for the first 6 bytes, it's unlikely to be a typo leading to the same exact first 6 bytes. If it doesn't match, it's *probably* not the right password (unless it's been tampered with right at the beginning of the encrypted content).

If the decrypted content passes the magic bytes test, but then the digest doesn't match, the data has probably been tampered with.

## Protocol version
Because there might be a V2 :-)

## Padding length and Padding
AES in both CBC and in XTS mode require the data length to be a multiple of the AES encryption block size (the AES algorithm being a [block cipher](https://en.wikipedia.org/wiki/Block_cipher)).

## Message Authentication Code (MAC)
The digest calculated on the plain content.

To see the full implementation, go check out the [go-encrypt](https://github.com/DanyJoly/go-encrypt) project on GitHub.

You can also see how I use it to do simple file encryption with the [go-filecrypt](https://github.com/DanyJoly/go-filecrypt) project.

# Limitations
## Not suitable for stream encryption
The protocol format puts the padding at the start of the encrypted content. That is impractical for stream encryption as you don't necessarily know the content length of a data stream in advance. If you do know the full size of the data though, appending the padding leads to a simpler implementation for both encryption and decryption.

If you need stream encryption, you could put the digest, the padding and its length at the end of the encrypted content. There are a few implementation bugs to be careful about when you'll read the data back though, assuming that you'll also support reading from a stream.

## MAC-then-encrypt or encrypt-then-MAC?
Just like VeraCrypt/TrueCrypt, I pack the digest along with the content. AES in XTS mode still being tamperable, it's possible in theory to tamper the data so that both the digest and the content match. From what I understand, this is not something likely, but if you have a way to hash the encrypted content and then keep the digest elsewhere securely, it's a better solution. It seemed rather impractical in the case of an encrypted file to pass along though so I don't think this would apply to this problem. I'm also assuming that if a better option existed, VeraCrypt/TrueCrypt would have done so already.

See a conversation about this point of contention [here](https://crypto.stackexchange.com/questions/202/should-we-mac-then-encrypt-or-encrypt-then-mac).

# References

AES: <https://en.wikipedia.org/wiki/Advanced_Encryption_Standard>

PBKDF2: <https://en.wikipedia.org/wiki/PBKDF2>

Argon2: <https://en.wikipedia.org/wiki/Argon2>

VeraCrypt: <https://veracrypt.fr>

TrueCrypt implementations: <http://blog.bjrn.se/2008/01/truecrypt-explained.html>

NIST Digital Security Guidelines - 2017: <https://pages.nist.gov/800-63-3/sp800-63b.html>
