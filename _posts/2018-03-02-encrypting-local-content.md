---
layout: post
title: Encrypting Local Content
comments: true
---

Last post was about general encryption tools such as password hash functions and content encryption. In this post, I'll show an example of local content encryption with a password inspired by what tools like [Bitlocker](https://en.wikipedia.org/wiki/BitLocker) or [VeraCrypt](https://www.veracrypt.fr) (a project that took over the now-defunct TrueCrypt) do.

Both those tools have more sophisticated needs than what we will cover here but AFAIK, the core steps are similar. For example, VeraCrypt can mount the file content as a drive and can even contain hidden encrypted drives in case the user is coerced in giving away the password. This brings additional complexity, but it's built upon the same foundation. There is this great article here about how [TrueCrypt](http://blog.bjrn.se/2008/01/truecrypt-explained.html), the ancestor of TrueCrypt, drives are encrypted (or at least were back in 2008).

# Security Note
I'll mention right away though that while this is here for informative purpose only. Don't rely on it for production systems.
, cryptography is really hard to get right.

because this uses AES with CBC encryption mode, this is not suitable for

# Let's go
The code covered in this example can be used to encrypt images or private content and be decrypted by someone with the right password.

Tools like Bitlocker and VeraCrypt use standard AES encryption, which I covered in my [previous blog post]({{ site.baseurl }}{% post_url 2018-02-26-cryptography-building-blocks-overview %}).

That by itself isn't sufficient to provide a useful solution though. Here's why:
* AES keys are fixed-length. AES-256 requires a 32 bytes key, which is quite hard to remember. Furthermore, the key is assumed to be as random as possible to cover the whole range of possibilities. User-selected keys are not random.
* AES doesn't protect against tampering.

I made a simple encryption-decryption Python package that I shared on GitHub as an example of the different steps required to do local content encryption.
