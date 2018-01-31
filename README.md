# Encryption is not Integrity


Don't we all remember the following common setup from our introductory security course?
Bob wants to send a secret message to Alice.
In order to obtain a key for encrypting the message, Alice and Bob first use Diffie-Hellman (DH) to exchange a fresh session key.
With this fresh session key, Bob symmetrically encrypts the message and sends it to Alice.
Carol volunteers to transmit the messages between Bob and Alice.
Here is the setup:
```
   Alice                            Carol                             Bob
     |                                |                                |
     |      DH values from Alice      |                                |
     |------------------------------->|      DH values from Alice      |
     |                                |------------------------------->|
     |                                |                             compute session key
     |                                |                                |
     |                                |        DH values from Bob      |
     |        DH values from Bob      |<-------------------------------|
     |<-------------------------------|                                |
compute session key                   |                             encrypt message
     |                                |                             with session key
     |                                |                                |
     |                                |       encrypted message        |
     |       encrypted message        |<-------------------------------|
     |<-------------------------------|                                |
decrypt message                       |                                |
with session key

```
One of the first things we learn in our introductory security course is that Carol could Man-in-the-Middle (MitM) the DH exchange to obtain session keys with Alice and Bob herself, while poor Alice and poor Bob still believe they are talking to each other.
The next thing an introductory security course teaches us is how to prevent this attack.
And here is how this article differs from an introductory security course:
Bob has the misconception that he can use encryption to prevent unauthorized modification.
As the title suggests, this does not work out well for Bob.
Neighbors, don't act like Bob.

Let us hear the story of Alice, Bob, and Carol.
There will be five attempts from Bob to transmit the encrypted message to Alice.
The protocol aborts prematurely if Carol could break the key before Bob has sent the message.

I hear our quality-conscious readers ask "Story?", surely followed by "PoC or GTFO!".
Dear esteemed reader, don't worry, the text you are reading right now was generated by \texttt{poc.py}.
You can follow along by \texttt{git clone https://github.com/diekmann/encryption-is-not-integrity.git}.
Let the story begin.



## Run 0: Prologue and Short recap of Diffie-Hellman

Alice and Carol are just returning from their introductory security course.
Bob, who also attended the lecture, walks over to Alice.
"If a message is encrypted, an attacker cannot read it and thus cannot modify it," Bob says to Alice.
Alice knows that encryption does not provide integrity and immediately wants to call bullshit on Bob's claim.
But she hesitates for a moment.
Bob won't appreciate an abstract explanation anyway.
"Let's see where this is going," she thinks and agrees to follow his explanation.
"I hope there will be code?" Alice responds.
Bob nods.


"Carol, come over, Bob is explaining crypto," Alice shouts to Carol.
Bob starts explaining, "Let's first create a fresh session key so I can send a secret message to you, Alice."
Alice agrees, this sounds like a good idea.
To make the scenario realistic, Alice makes sure that neither Bob nor Carol can see her screen.
She opens her python3 shell and is about to generate some DH values.
"We need a prime $p$ and a generator $g$," Alice says.
"607 is a prime", Bob says with wikipedia open in his browser.
Alice, hoping that Bob is joking about the size of his prime, suggests the smallest prime from RFC 3526 as an example:
```
      FFFFFFFF FFFFFFFF C90FDAA2 2168C234 C4C6628B 80DC1CD1
      29024E08 8A67CC74 020BBEA6 3B139B22 514A0879 8E3404DD
      EF9519B3 CD3A431B 302B0A6D F25F1437 4FE1356D 6D51C245
      E485B576 625E7EC6 F44C42E9 A637ED6B 0BFF5CB6 F406B7ED
      EE386BFB 5A899FA5 AE9F2411 7C4B1FE6 49286651 ECE45B3D
      C2007CB8 A163BF05 98DA4836 1C55D39A 69163FA8 FD24CF5F
      83655D23 DCA3AD96 1C62F356 208552BB 9ED52907 7096966D
      670C354E 4ABC9804 F1746C08 CA237327 FFFFFFFF FFFFFFFF
```

This is a 1536-bit prime.
Alice notes fascinated, "this prime has $\pi$ in it!"
According to the RFC, the prime is $p = 2^{1536} - 2^{1472} - 1 + 2^{64} \cdot (\lfloor 2^{1406} pi \rfloor + 741804)$.
Alice continues to think aloud, "Let me reproduce this. Does that formula actually compute the prime? Python3 integers have unlimited precision, but $\pi$ is not an integer."
"Python also has floats," Bob replies.
Probably Bob was not joking when he suggested 607 as large prime previously.
It seems that Bob has no idea what `large' means in cryptography.
Meanwhile, using
```python
>>> import decimal
```
Alice has reproduced the calculation.
By the way, the generator $g$ for said prime is conveniently 2.

A small refresher on DH follows:

```
=== BEGIN SNIPPET RFC 2631 ===
2.1.1.  Generation of ZZ

   [...] the shared secret ZZ is generated as follows:

     ZZ = g ^ (xb * xa) mod p

   Note that the individual parties actually perform the computations:

     ZZ = (yb ^ xa)  mod p  = (ya ^ xb)  mod p

   where ^ denotes exponentiation

         ya is party a's public key; ya = g ^ xa mod p
         yb is party b's public key; yb = g ^ xb mod p
         xa is party a's private key
         xb is party b's private key
         p is a large prime
=== END SNIPPET RFC 2631 ===
```


Alice takes the initiative, "Okay, I generate a secret value $\mathit(xa)$, compute $\mathit{ya} = g^\mathit{xa} \bmod p$ and send to you $\mathit{ya}, g, p$. This is also how we did it in the lecture."
Bob then has to choose a secret value $\mathit(xb)$, compute $\mathit{yb} = g^\mathit{xb} \bmod p$ and send $\mathit{yb}$ back to Alice, so she can compute $\mathit{ZZ}_a$.
Bob then uses the key $\mathit{ZZ}_b$ he computed to encrypt a message and send it to Alice.
Since $\mathit{ZZ}_b = \mathit{ZZ}_a$, Alice can decrypt the message.

This is what Alice and Bob plan to do:
```
   Alice                            Carol                             Bob
xa = random()                         |                                |
ya = pow(g, xa, p)                    |                            xb = random()
     |                                |                            yb = pow(g, xb, p)
     |           (ya, g, p)           |                                |
     |------------------------------->|           (ya, g, p)           |
     |                                |------------------------------->|
     |                                |                            ZZ_b = pow(ya, xb, p)
     |                                |                                |
     |                                |               yb               |
     |               yb               |<-------------------------------|
     |<-------------------------------|                                |
ZZ_a = pow(yb, xa, p)                 |                            ciphertext =
     |                                |                            Enc(ZZ_b, message)
     |                                |           ciphertext           |
     |          ciphertext            |<-------------------------------|
     |<-------------------------------|                                |
Dec(ZZ_a, ciphertext)
= message

```
"Let's go then," Bob says.
"Wait," Alice intervenes, "DH is only secure against passive attackers. An active attacker could MitM our exchange."
Alice and Bob look at Carol, she smiles.
Alice continues, "What did you say in the beginning?"
"Right," Bob says, "we must encrypt our DH values, so Carol cannot MitM us."
Fortunately, Alice and Bob have 4096-bit RSA keys and securely distributed their public keys beforehand.

"Okay, what should I do?" Alice asks.
Besides, Alice knows exactly what to do, but Bob's stackoverflow-driven approach to crypto may prove useful in the course of this story.
Bob types into Alice's terminal
```python
>>> import Crypto.PublicKey.RSA
>>> def RSA_enc(k_pub, msg):
...     return k_pub.encrypt(msg, None)[0]
```
He comments, "We can ignore this None and only need the first value from the tuple. Both exist only for compatibility."
Bob is right about that and we now have a convenient textbook RSA encryption function at hand.


## Run 1: RSA-Encrypted textbook DH in one line of python


Now Alice and Bob are ready for their DH exchange.
In contrast to their original sketch, they will encrypt their DH values with RSA.
Alice generates:
```python
>>> xa = int.from_bytes(os.urandom(192), byteorder='big')
>>> ya = pow(g, xa, p)
```
and sends
```python
>>> RSA_enc(k_Bob_pub, (ya, g, p))
```
Alice sends 806609c69f1eea93... [504 bytes omitted]
How does Alice send the message?
She hands it over to Carol.
Carol starts fiddling around with with the data.
"What are you doing?" Bob shouts.
Alice tries to calm him down, "It is encrypted, those were your words. Carol will deliver the message to you."
Bob calms down and nods.
Alice winks at Carol, she knows what to do.
Carol forwards 7e155f2f87801a2e... [504 bytes omitted]
Bob decrypts with his private RSA key, parses ya, g, p from the message, and computes
```python
>>> xb = int.from_bytes(os.urandom(192), byteorder='big')
>>> yb = pow(g, xb, p)
>>> ZZ_b = pow(ya, xb, p)
```
and sends
```python
>>> RSA_enc(k_Alice_pub, yb)
```
Bob sends 08f0cbf74bfc5d64... [504 bytes omitted]
Carol forwards a different message.
Alice performs her part to finish the DH handshake.
Carol exclaims, "The key is 1!"
Bob and Alice check.
Carol is right.
How can Carol know the established keys?
Bob is right about one thing, the DH values were encrypted, so a trivial textbook DH MitM attack does not work since Carol cannot get the ya and yb values.
But she doesn't need to.
This is what happened so far:
```
   Alice                            Carol                             Bob
     |                                |                                |
     |   RSA(k_Bob_pub, (ya, g, p))   |                                |
     |------------------------------->|                                |
     |                                |   RSA(k_Bob_pub, (1, g, p))    |
     |                                |------------------------------->|
     |                                |                           RSA decrypt
     |                                |                           ZZ_b = pow(1, xb, p)
     |                                |                                |
     |                                |                                |
     |                                |      RSA(k_Alice_pub, yb)      |
     |                                |<-------------------------------|
     |       RSA(k_Alice_pub, 1)      |
     |<-------------------------------|
RSA decrypt
ZZ_a = pow(1, xa, p)
```

The prime $p$, the generator $g$, and the public keys are public knowledge, also known to Carol (check your textbook, neighbor).
Consequently, Carol can encrypt DH values, but she cannot read the ones from Alice and Bob.
Bob computes the shared DH key as $\mathit{ya}^\mathit{xb} \bmod p$, where Carol supplied $1$ for $\mathit{ya}$.
Carol can be sure that Bob will compute a shared key of $1$, she doesn't need to know any encrypted values.
Same goes for the exchange with Alice.

"No No," Bob shouts, "these values are not allowed in DH."
Alice checks RFC 2631 and quotes: <<The following algorithm MAY be used to validate a received public key y [...] Verify that y lies within the interval [2,p-1]. If it does not, the key is invalid.>>
Bob replies, "So y = 1 is clearly invalid, you must not do this Carol."
Alice objects, "The check is optional, see this all-caps MAY there?"
But Bob feels certain that he is right and insists, "Any library would reject this key!"


## Run 2: RSA-Encrypted textbook DH using parts of the OpenSSL library

"Sure, we'll give it a try." Alice responds.
She sticks to her old code because the RFC clearly states the check optional, but Bob can reject the weak values.

Alice sends 6dd7b4286e49e3db... [504 bytes omitted]
Carol, testing the same trick again, forwards 7e155f2f87801a2e... [504 bytes omitted]
Bob now uses \texttt{pyca/cryptography} with the openssl backend to do the DH computation.
Maybe just doing `ZZ_b = pow(ya, xb, p)` was too simple?
Let's see what happens when we use some part of the OpenSSL library (wrapped by \texttt{pyca/cryptography}) to perform the same computation.
A word of clarification: The OpenSSL library is only used to implement the DH part on Bob's side, the exchange is not tunneled over TLS.
The RSA-part remains unchanged.

```python
>>> from cryptography.hazmat.primitives.asymmetric import dh
>>> from cryptography.hazmat.backends import openssl
>>> pn = dh.DHParameterNumbers(p, g)
>>> parameters = pn.parameters(openssl.backend)
>>> xb = parameters.generate_private_key()
>>> # feed ya to the openssl library backend
>>> alice_public_key = dh.DHPublicNumbers(ya, pn).public_key(openssl.backend)
>>> assert alice_public_key.key_size == 1536 # 1536-bit MODP group of our prime
>>> yb = xb.public_key().public_numbers().y
>>> ZZ_b = xb.exchange(alice_public_key)
```
And indeed, the last line aborts with the exception `ValueError: Public key value is invalid for this exchange.'
Alice and Bob abort the handshake.
This is what happened so far:
```
   Alice                            Carol                             Bob
     |                                |                                |
     |   RSA(k_Bob_pub, (ya, g, p))   |                                |
     |------------------------------->|                                |
     |                                |   RSA(k_Bob_pub, (1, g, p))    |
     |                                |------------------------------->|
     |                                |                           RSA decrypt
     |                                |                           With (ya = 1, g, p)
     |                                |                           using openssl.backend to
     |                                |                           compute ZZ_b ...
     |                                |                           raise ValueError

```

"Now you must behave, Carol. We will no longer accept your MitMed values. Now that we prohibit the two bad DH values and everything is encrypted, we are 100% secure," Bob says.

## Run 3: RSA-Encrypted textbook DH using parts of the OpenSSL library and custom Primes


Alice and Bob try the handshake again. Carol cannot send $ya = 1$ because Bob will detect it and abort the handshake.
Alice sends 48dd9752297b0710... [504 bytes omitted]
But Carol knows the math.
She chooses a nice prime $\mathit{pc}$ and computes a random, valid $\mathit{yc}$ value.
```python
>>> pc = 2**1536 - 1
>>> xc = int.from_bytes(os.urandom(192), byteorder='big')
>>> yc = pow(g, xc, pc)
```
Well, $pc$ isn't actually a prime.
Let's see if OpenSSL accepts it.
Reliably testing for primality is expensive,\footnote{Common primality tests are probabilistic and relatively fast, but can err. Deterministic primality tests in polynomial time exist. Note that DH does not need an arbitrary prime and some $g$, but the generator should generate a not-too-small\textsuperscript{TM} subgroup.} chances are good that the prime gets waved through.
Carol forwards 60c916ee273b02de... [504 bytes omitted]
After RSA decryption, Bob's code happily accepts all values.
Bob sends 836546fbfb398219... [504 bytes omitted]
Alice still thinks that the RFC 3526 prime is used.
Carol just forwards random plausible values to Alice, but she won't be able to MitM this key.
Carol forwards 361bbd0dd23939a4... [504 bytes omitted]
The DH key exchange is successfully completed.
Now Bob can use the key $\mathit{ZZ}_b$ established with DH to send an encrypted message to Alice.
```python
>>> iv = os.urandom(16)
>>> aeskey = kdf128(ZZ_b) # squash the key to 128 bit
>>> ct = aes128_ctr(iv, aeskey, b'Hey Alice! See, this is perfectly secure now.')
>>> wire = "{},{}".format(hexlify(iv).decode('ascii'), hexlify(ct).decode('ascii'))
```
Bob sends the iv and the ciphertext message5b736ed615a200b911778c9b4b3f3f4f,c7f58f39a8614d6fec8e51bb1df7fd6edc8df349968d24227dcf2fa9af01455a6c74e0a0f0af58332f631c3bb8.
In summary, this is what happened so far:
```
   Alice                            Carol                             Bob
     |                                |                                |
     |   RSA(k_Bob_pub, (ya, g, p))   |                                |
     |------------------------------->|                                |
     |                                |  RSA(k_Bob_pub, (yc, g, pc))   |
     |                                |------------------------------->|
     |                                |                           RSA decrypt
     |                                |                           using openssl.backend
     |                                |                           ZZ_b = pow(yc, xb, pc)
     |                                |                                |
     |                                |                                |
     |                                |      RSA(k_Alice_pub, yb)      |
     |                                |<-------------------------------|
     |    RSA(k_Alice_pub, garbage)   |                                |
     |<-------------------------------|                           ciphertext =
RSA decrypt                           |                           Enc(ZZ_b, message)
ZZ_a = garbage2                       |           ciphertext           |
     |                                |<-------------------------------|

```
Carol chose a great `prime' $\mathit{pc}$ and knows the key is broken: Only one bit is set!
She can just brute force all possible keys, the one that decrypts the ciphertext to printable ASCII text is most likely the correct key.
```python
>>> iv, ct = map(unhexlify, wire.split(','))
>>> for i in range(1536):
...     keyguess = 2**i
...     msg = aes128_ctr(iv, kdf128(keyguess.to_bytes(192, byteorder='big')), ct)
...     try:
...         if not all(c in string.printable for c in msg.decode('ascii')):
...             continue
...     except UnicodeDecodeError: #not ASCII
...         continue
...     break

```
The brute-forced key is 247330401473104534060502521019647190035131349101211839914063056092897225106531867170316401061243044989597671426016139339351365034306751209967546155101893167916606772148699136, or in hex '\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00' (exactly one bit set). Carol is correct.
She immediately shouts out the message `Hey Alice! See, this is perfectly secure now.'
Bob is depressed.
"Why doesn't this code work?", he asks.
"Probably DH is not strong enough and we need to use elliptic curve DH?", he conjectures.
"Maybe Carol even has a quantum computer hidden in her pocket, let me find a post-quantum replacement for Diffie-Hellman, ..." he continues.
Carol interferes, "The same ideas of my attack also apply to ECDH or a post-quantum drop-in replacement with the same properties. Don't waste your time on this line of thought. If you cannot use textbook DH, ECDH (or the post-quantum candidates) won't help."


## Run 4: Textbook DH signed with textbook RSA


Alice tries to help without hurting Bob's ego, "Maybe RSA encryption does not help, but can we use RSA differently? Remember, encryption itself does not not provide integrity."
"Of course," Bob replies, "we need to sign the DH values. And signing with RSA is just encryption with the private key."
"Don't forget the padding," Alice is trying to help, but Bob doesn't understand and just ignores Alice's comment.
Bob immediately codes:
```python
>>> import Crypto.PublicKey.RSA
>>> def RSA_sign(k_priv, msg):
...     # ignore the compatibility parameters
...     return k_priv.sign(msg, None)[0]
>>> def RSA_verify(k_pub, msg, signature):
...     # ignore the compatibility parameters
...     return k_pub.verify(msg, (signature, None))
```
Again, Bob is right about ignoring the compatibility parameters.
However, Carol sneers as Bob completely ignored Alice's comment about padding.

"Let's hardcode the prime $p$ and generator $g$ for simplicity and switch back to the trivial non-OpenSSL implementation." Alice suggests and everybody agrees.
This simplifies the DH exchange as now, only $y$ and the signature of $y$ will be exchanged.
Alice only sends the following in the first step:
```python
>>> "{},{}".format(ya, RSA_sign(k_Alice_priv, ya))
```
Alice sends 3104b3fa491483bf...[184 bytes of y omitted],6fa3bbd123df529e...[504 bytes of signature omitted]
Carol just forwards 1,1
Bob parses the values, verifies the signature correctly and performs his step of the DH exchange.
```python
>>> ya, signature = map(int, wire.split(','))
>>> if not RSA_verify(k_Alice_pub, ya, signature):
>>>     print("Signature verification failed")
>>>     return 'reject'
[...]
>>> return "{},{}".format(yb, RSA_sign(k_Bob_priv, yb))
```
Bob sends 5272820601a4245d...[184 bytes of y omitted],47e1406c0f9eae79...[504 bytes of signature omitted]
Carol just forwards 1,1
Alice smiles as she receives the values.
Nevertheless, she performs the signature verification professionally.
Both, the signature check at Bob and the signature check at Alice were successful and Alice and Bob agreed on a shared key.
This is what happened so far, where \texttt{RSA} corresponds to \texttt{RSA_sign} as defined above:
```
   Alice                            Carol                             Bob
     |                                |                                |
     |   ya, RSA(k_Alice_priv, ya)    |                                |
     |------------------------------->|                                |
     |                                |               1,1              |
     |                                |------------------------------->|
     |                                |                           RSA_verify(k_Alice_pub, 1, 1)
     |                                |                           ZZ_b = pow(1, xb, p)
     |                                |                                |
     |                                |                                |
     |                                |     yb, RSA(k_Bob_priv, yb)    |
     |                                |<-------------------------------|
     |               1,1              |
     |<-------------------------------|
RSA_verify(k_Bob_pub, 1, 1)
ZZ_a = pow(1, xa, p)

```
Carol exclaims "The key is 1!"
Bob is all lost, "How could this happen again? I checked the signature!"
"Indeed," Carol explains, "but you should have listened to Alice's remark about the padding.
RSA signatures are not just the textbook RSA operation with the private key.
Plain textbook RSA is just $\mathit{msg}^d \bmod N$, where $d$ is private.
Guess how I could forge a valid RSA private key operation without knowledge of $d$ if I may choose $\mathit{msg}$ freely?"
Bob looks at Carol as if all his textbook knowledge about RSA crumbles.
"Can Carol break RSA? What is the magic math behind her attack?", he asks himself.
Carol helps, "$1^d \bmod N = 1$, for any $d$. Of course I did not break RSA. The way you tried to use RSA as a signature scheme is just not existentially unforgeable.
Paddings, or signature schemes, exist for a reason."
By the way, the RSA encryption without padding used in the previous runs is also dangerous.\footnote{Use OAEP!}


## Run 5: Textbook DH signed with RSASSA-PSS

Bob gets some help to get the sign and verify functions replaced.
```python
>>> from cryptography.hazmat.primitives import hashes
>>> from cryptography.hazmat.primitives.asymmetric import padding
>>> def RSA_sign(k_priv, msg):
>>>     return k_priv.sign(
...         msg,
...         padding.PSS(
...             mgf=padding.MGF1(hashes.SHA256()),
...             salt_length=padding.PSS.MAX_LENGTH
...         ),
...         hashes.SHA256()
...     )
```
The RSA_verify function is replaced accordingly.

Now Alice and Bob can try their handshake again.
Alice sends 99bf35375a9a9e94...[184 bytes of y omitted],88bb02530abd586f...[504 bytes of signature omitted]
Carol forwards the message unmodified.
Bob looks at Carol suspiciously.
"I cannot modify this without breaking the signature," Carol replies.
"Probably the DH prime is a bit too small for the future; Logjam predicts 1024-bit breakage. Maybe you could use fresh DH values for each exchange or switch to ECDH to be ready for the future, ... 
But I'm not aware of any attack I could carry out on my slow laptop against your handshake for now." Carol concludes.

Bob sends f4822577fdc27c7d...[184 bytes of y omitted],4025364e27006985...[504 bytes of signature omitted]
Carol forwards the message unmodified.
Finally, Alice and Bob established a shared key and Carol does not know it.

```
   Alice                            Carol                             Bob
     |                                |                                |
     |   ya, RSA(k_Alice_priv, ya)    |                                |
     |------------------------------->|                                |
     |                                |   ya, RSA(k_Alice_priv, ya)    |
     |                                |------------------------------->|
     |                                |                           RSA_verify(k_Alice_pub, ...)
     |                                |                           ZZ_b = pow(ya, xb, p)
     |                                |                                |
     |                                |                                |
     |                                |     yb, RSA(k_Bob_priv, yb)    |
     |                                |<-------------------------------|
     |    yb, RSA(k_Bob_priv, yb)     |
     |<-------------------------------|
RSA_verify(k_Bob_pub, ...)
ZZ_a = pow(yb, xa, p)

```
To complete the scenario, Bob uses the freshly established key to send an encrypted message to Alice.
```python
>>> iv = os.urandom(16)
>>> aeskey = kdf128(ZZ_b) # squash the key to 128 bit
>>> ct = aes128_ctr(iv, aeskey, b'Hey Alice! See, this is perfectly secure now.')
>>> wire = "{},{}".format(hexlify(iv).decode('ascii'), hexlify(ct).decode('ascii')
```
Bob sends the iv and the ciphertext message 65c04be185d2f42df90cb3c8f1a418f9,c02cc58d935bc8fa1b2659b8dcb1be6914d9b4eb57aea84ee38b61fe5a0b8cce25a200235d8492f6e028d2fe19.
Carol realizes that the ciphertext has exactly the same length as the message sent by Bob in run 3 when he thought he had established a key with Alice.
Carol forwards 65c04be185d2f42df90cb3c8f1a418f9,cd27dfdfab47d5f011695982caf4fc26149194f603a2bc1cfa9a6ab6
Alice reads out loud the message she received and decrypted: `Encryption is not Integrity.'
Bob shouts, "This is not the message! How can this happen? Did Carol break AES-CTR?"
Alice and Carol answer simultaneously, "AES-CTR is perfectly secure encryption, but Encryption is not Integrity."

