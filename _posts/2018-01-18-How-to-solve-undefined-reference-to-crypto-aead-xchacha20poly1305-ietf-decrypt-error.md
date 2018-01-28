---
layout: post
title: "How to solve Undefined reference to `crypto_aead_xchacha20poly1305_ietf_decrypt' Error"
date: 2018-01-16 23:10:01 +0800
tags: ["Ansible"]
---


If you encounter the error like this：
```
xxx: In function `crypto_derive_key':undefined reference to `crypto_pwhash'
xxx: In function `cipher_aead_decrypt':undefined reference to `crypto_aead_xchacha20poly1305_ietf_decrypt'
xxx: In function `cipher_aead_encrypt':undefined reference to `crypto_aead_xchacha20poly1305_ietf_encrypt'
collect2: error: ld returned 1 exit status
make[2]: *** [xxx] Error 1
make[2]: Leaving directory `xxx'
make[1]: *** [xxx] Error 1
make[1]: Leaving directory `xxx'
make: *** [all] Error 2
```

Try:
1.Remove old libsodium from system,run `apt-get purge libsodium-dev`, and compile again
2. Try to add libsodium path along with ./configure command like `./configure –with-sodium=xxx` 
