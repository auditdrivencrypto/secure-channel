## Protocol Draft 2

version 2 also supports property [#16](./properties.md#16-mitmwrong-number-cannot-learn-or-confirm-keys)

### dependency: crypto_box

This protocol depends on the `crypto_box` primitive as implemented in [nacl](http://nacl.cr.yp.to/box.html).
Unfortunately, that primitive is not very well documented. I will attempt to explain it's properties here. The security model of this primitive is only briefly covered on the [documentation](http://nacl.cr.yp.to/box.html) site so this is based mainly on my reading of the code. Please do check my reasoning here, and make an issue if you think I missed something.

`crypto_box` takes a message, a nonce, a public key and a private key.
`crypto_box(message, nonce, alice.public_key, bob.private_key)` which is decrypted by
`crypto_box_open(boxed, nonce, bob.public_key, alice.private_key)`.
The message is encrypted with salsa20 cipher, and authenticated with poly1305. There is no length delimitation so if you wish to transmit this message it must be framed, or have a fixed size, the other party requires the same nonce in order to perform the decryption so that must be provided some way (i.e. either by sending it along with the message, or by having a protocol for determining the next nonce)

Although it's described as Bob _encrypting to_ Alice ("Bob boxes the message to Alice") the encryption is not directional, and either Bob _or_ Alice can decrypt the message. This is because it derives a shared key in the manner of a diffie-helman key exchange, _not_ by encrypting a key to Alice's pub key (which would be an operation that Bob could not reverse). This has a surprising property if this is used as an authentication primitive: If an attacker gains Bob's private key, and knows Alice's key then they can not only impersonate bob to Alice (or anyone), but surprisingly they can impersonate _anyone_ to Bob (provided they know that public key)! 

This would make a compromise of his private key a decidedly schizophrenic experience for Bob! Although to other parties, Bob suddenly acting weird would be simple enough to diagnose - Bob has been hacked - but Bob may instead experience _everyone he knows_ suddenly going schizophrenic. This could potentially be more destructive than merely impersonating Bob. Hopefully loosing control of one's private keys is an extremely unlikely event, but the antics of bitcoin has certainly shown this is possible via a variety of avenues if attackers are sufficiently motivated. If it's reasonable to design a protocol to be forward secure (not leak information if keys are compromised) then it's reasonable to make other aspects of the protocol fail safely in the case of key compromise.

Therefore, my conclusion is that `crypto_box` is not suitable as a _user authentication primitive_, and signatures should be used instead (though, the signatures may be inside a `crypto_box` for privacy). 

### protocol description

Alice: Hi call me Andy
> Alice generates a temporary key, Andy, and sends it to (the server she thinks is) Bob

Bob: Hi, call me Betty
> Bob generates a temporary key, Betty, and sends it back to whoever connected (doesn't know it's Alice yet)

Alice now sends a secure greeting to Bob that will prove her identity, but only to Bob.

Alice:
```
  //this is most complicated bit, easier to specify it as pseudocode.

  //encrypt the entire message to Betty (bob's temp key)
  box[Andy->Betty](
    //reveal long term identity (Alice) but only to Bob! (via a box)
    //this is inside the outer box to get forward security (eavesdropper will have outside box only)
    //Boxed from Andy, since Bob doesn't know it's Alice yet!
    box[Andy->Bob]([
      //send Alice's private key
      Alice.public_key,
      //creates a 3rd public key
      (Aaron = createKey()).public_key
      //sign the hash of the first two messages, this proves to bob this packet was created by holder
      //of Alice's key, and cannot be a replay or mitm attack because Bob knows Betty's key is brand new.
      sign(
        hash(Andy.public_key + Betty.public_key + Aaron.public_key),
        Alice.private_key
      )
    ])
  )
```

> Alice has now authenticated to Bob. Bob knows that Alice intended to connect to him. And since he opened the box, and found Alice's key, and a signature from Alice. Since Alice also signed something that didn't exist just a moment ago (Betty.public_key) then he knows it's not a replay attack. Since a man in the middle cannot know the temp key he generated (Betty _and_ Andy) then he knows there cannot be a mitm who is decrypting the packets (TODO: more discussion about what a mitm can achieve).

If the call was not for him (wrong number) then he cannot open the inner box, so the connection must be dropped (and Bob will not learn Alice's identity). If he doesn't wish to talk to Alice, then the call should also be dropped. This means a disconnection at this point does not confirm the server's identity. 

Alice sends a new key (Aaron) which will be used for the remainder of the session - it might not be strictly necessary to use a third key, but I liked the idea of making the proof of security via cryptography instead of by protocol state.

If Bob does decide to connect with Alice, then he responds with another message containing a new key (Barbara) and signing it

Bob: 

```
//encrypt to temp id...
box[Betty->Andy](
  //... a message eve can't see...
  box[Bob->Alice]([
    //containing a new key
    Barbara.public_key
    //with proof this message was just created by Bob!
    //note: also signing Aaron.public_key proves to alice that bob did actually decrypt the hello packet.
    sign(hash(hello + Barbara.public_key + Aaron.public_key), Bob.private_key)
  ])
)
```

Now Alice and Bob are mutually authenticated! Bob knows he's talking to Alice, and Alice knows she is talking to Bob. _as far as I have determined, no weird edge cases_. Of course, if your key is compromised, then someone can impersonate you, this is to be expected, and key revocation should be solved in another part of the cryptosystem.

the rest of the session is encrypted with Aaron/Barbara. Even the existence of these keys is a secret from both an eavesdropper or a man in the middle!

This design realizes _all_ the [desirable secure channel properties](https://github.com/ssbc/scuttlebot/wiki/desirable-properties-for-a-secure-channel)


