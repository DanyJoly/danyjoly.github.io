---
layout: post
title: Cryptography Building Blocks Overview
comments: true
---

Over the past few months, I got into situations where a better understanding of cryptography would have saved me some time.

Simple things like *what's the best way to store a user's password?*, or *how can I encrypt content in my app for safe storage?*

Of course, the best advice with cryptography security is to use solution that someone else was paid a lot of money to create. Cryptography is one of those fields where even a solution that has been crafted by world-class experts, standardized, then used successfully for many years, can be vulnerable and broken [15 years later by a clever hacker](https://blog.cloudflare.com/padding-oracles-and-the-decline-of-cbc-mode-ciphersuites/).

Still, understanding how some of those solutions work is fun. It also helps to understand how they work so that you can assess if they provide the protection you need. For example, is encryption sufficient, or do you also need your data to be tamper proof?

In this post, I'll only cover certain basic building blocks used to create secure protocols and software. It's already a longish read so I'll wait for later posts to cover how they can be used together to create secure software solutions.

Here we go!

# Hashing
At its core, a hash function is simply an algorithm that takes a data input of arbitrary length and output a value of fixed length. The output is called the hash or the digest.

Not all hash functions have the same goals so they cannot be used interchangeably.

## Protection against Data Corruption

Many hash functions like [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) and [MD5](https://en.wikipedia.org/wiki/MD5) are used to detect *accidental* changes to data. Usually, the longer the hash, the better protection against data corruption. For Example CRC-32 (32 bits hash) has a lower chance of collision than CRC-16 (16 bits hash).

A good error-detecting hash will be quick to compute and has a low chance of random collision. That is, different data should have different hash values.

CRC-32 is used by the Ethernet protocol for example to detect data corruption in transmission. If the CRC value doesn't match, the package is retransmitted by the sender. It's something I've implemented myself back in college to detect data corruption during transfers over a noisy wire between an embedded system and a laptop. The corruption check and the data recovery code in place saved our experiment.

While very useful, those hash functions are not related to cryptography. That's usually because in cryptography, we fear not random changes to data, but a malicious attacker that will craft the data to create a collision. The requirements are much more strict.

## Cryptographic Hash Functions: Protection against Data Tampering

There is a whole class of hash functions named *cryptographic hash function* that can be very useful to detect data tampering or even create digital signatures when used properly.

Here are some properties that strong cryptographic hash function share:
* it's reasonably fast to calculate;
* it's a one-way function: it's not possible to reverse engineer the original data from the hash;
* small changes in the input generate large changes in the output so that the two inputs seem unrelated;
* it’s computationally impractical to generate data that also have the same hash value. In cryptography, this means that only brute force is a reasonable strategy and average computation time is prohibitive (think millions of years).

The [SHA](https://en.wikipedia.org/wiki/Secure_Hash_Algorithms) family of hash functions was developed with these goals in mind. They are reasonably fast to compute and are very hard to tamper with. While SHA-1 is still sometime used, it's not considered safe. Nowadays, we use SHA-2 and SHA-3. Notably in the SSL/TLS protocol, which essentially means the whole Internet currently depends on them!

Like with regular hash functions, the longer the resulting hash, the stronger the hash function (albeit at a performance cost).

It's interesting to know that [MD5](https://en.wikipedia.org/wiki/MD5) was originally designed as a cryptographic hash function, but too many vulnerabilities were found over the years so now, while it's unsafe against data tampering, it's still useful as a strong protection against random data corruption.

## Password Hash Functions

So say you know from news like [this](http://money.cnn.com/2012/07/12/technology/yahoo-hack/index.htm) that storing passwords in plain text is not the greatest idea. The problem is compounded by the fact that users do reuse their passwords. Even if your service is only something trivial like a forum, you could still be storing the key to their bank account if they reused their password. [Attackers certainly know this](https://techcrunch.com/2016/06/16/github-accounts-targeted-in-password-reuse-attack/).

Some people though, "wow I could *hash* the password and then when my users enter their password, I'll compare the hash!"

It's a neat idea, but since cryptographic hash such as the SHA family are fast to compute, and since passwords are very short, they are trivial to break by brute force. So, [don't store passwords with a cryptographic hash function](http://securityuncorked.com/2012/06/how-to-crack-your-own-linkedin-password-hash/). Better solutions exist.

Password hash functions are similar to cryptographic hash functions, but they are very computationally expensive to compute to protect against brute force attacks. They main objective is to help store and/or validate passwords securely.

One thing all password hash functions share is that their complexity can be adjusted. As computing power increases, they can be made more expensive. It's important to follow up to date best practices for those settings and review them from time to time. It also means that these passwords likely won't be safe forever. It's only a matter of time before computing power catch up with you. I have often seen 30 years as a good default achievable goal if you used proper settings and no vulnerabilities were found in the hash algorithm.

Password hash function are still vulnerable to things like [rainbow table attacks](https://en.wikipedia.org/wiki/Rainbow_table). A fancy name to essentially mean that wealthy attackers can use huge computer farms to precompute hash values and store it in a table to be used in an attack.

Protection against these types of attacks are usually to make the hash function as expensive as possible while still providing an acceptable level of performance and mostly, to use a password [salt](https://en.wikipedia.org/wiki/Salt_(cryptography)) to make the attack impractical.

Some password hash functions like [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) and [bcrypt](https://en.wikipedia.org/wiki/Bcrypt) have been around for 15+ years and are still going strong.

In recent years, custom FPGA and GPU-based password hashing solutions have been built to increase attacker's computing power by multiple orders of magnitude. To mitigate against these new tools, a new generation of password hash functions have been developed. [Argon2](https://en.wikipedia.org/wiki/Argon2) and [scrypt](https://en.wikipedia.org/wiki/Scrypt) are recent examples. On top of a large demand on CPU resources, they also require large memory resources, which is not something as easy to optimize with current GPU-based solutions.

While these functions are likely stronger than their predecessors, they have not been as battle-tested as PBKDF2 and bcrypt. I guess it's always possible to do a half-half and apply PBKDF2 or bcrypt first, then feed the hash to Argon2 or scrypt.

# Encryption - Protection Against Eavesdropping

Encrypting data means that no one can read it unless they have the key. It's an old and very specialized field with numerous techniques to break new cyphers. Even when using the best cypher algorithms, I noticed that it's very easy to use the wrong combination or settings and have your ass handed to you. Even well known world-class protocols [still evolve as new attacks are discovered](https://www.gracefulsecurity.com/tls-ssl-vulnerabilities/).

## Symmetric keys encryption
Functions that can encrypt and decrypt content using the same key. To be useful in cryptography, decrypting the content without the key must be computationally prohibitive.

The key must remain secret, otherwise, the content can be decrypted by whoever has the key.

Examples of asymmetric keys encryption: [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) (the most commonly used, NSA approved), Serpent and Twofish (two AES contenders), DES (unsafe, deprecated ancestor of AES).

It's important to know that AES encryption is by itself insufficient for secure communication: the attacker can still modify the encrypted data without being detected. It doesn't identify the sender either. That's another reason why implementing your own encryption solution is so tricky.

More information:
How AES works: http://www.moserware.com/2009/09/stick-figure-guide-to-advanced.html

## Asymmetric keys encryption
Unlike symmetric key encryption, this one has different keys for encrypting and decrypting data. To be useful in cryptography, a key must not be discoverable from the other key in a practical manner (computationally prohibitive).

When people talk about private and public key, this is what they are referring to. If the write key is private, the sender can encrypt data so that the receivers can authenticate the sender. Alternatively, the read key can be private so that anyone can send messages that can be read only by a single person.

While asymmetric key encryption has many usages, it’s also very computationally expensive so large exchange of data with them is impractical. This is why SSL/TLS uses asymmetric key encryption for the initial handshake, but then generate and exchange a symmetric key for the rest of the encrypted session.

Examples of asymmetric keys encryption: [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem))

# Next Posts

All right that's it for now. While all of this information is sometime very theoretical, in future posts I'll analyze specific recipes using these tools to do things such as file encryption and digital certificates. Stay tuned!
