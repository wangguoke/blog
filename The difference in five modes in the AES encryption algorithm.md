Recently, I did some work with Sawada-san on the TDE. So I studied on the encryption algorithm.
So far, I study five modes in the AES. In this document, I will introduce the difference in the five kinds of mode.

### General
The block ciphers are schemes for encryption or decryption where a block of plaintext is treated as a single block and is used to obtain a block of ciphertext with the same size.
Today, AES (Advanced Encryption Standard) is one of the most used algorithms for block encryption. It has been standardized by the NIST (National Institute of Standards and Technology) in 2001,
in order to replace DES and 3DES which were used for encryption in that period.
The size of an AES block is 128 bits, whereas the size of the encryption key can be 128, 192 or 256 bits.
Please note this, there is three length in the key, but the size of the encryption block always is 128 bits.
Block cipher algorithms should enable encryption of the plaintext with size which is different from the defined size of one block as well. We can use some algorithms for padding block when the plaintext is not enough a block, like PKCS5 or PKCS7, it also can defend against PA attack, if we use ECB or CBC mode. Or we can use the mode of AES which support a stream of plaintext, like CFB, OFB, CTR mode.
Now let's introduce the five modes of AES.
* ECB mode: Electronic Code Book mode
* CBC mode: Cipher Block Chaining mode
* CFB mode: Cipher FeedBack mode
* OFB mode: Output FeedBack mode
* CTR mode: Counter mode

The attack mode:
*  PA: Padding attack
*  CPA: Chosen Plaintext Attack
*  CCA: Chosen Ci

### ECB Mode
The ECB (Electronic Code Book) mode is the simplest of all. Due to obvious weaknesses, it is generally not recommended.
A block scheme of this mode is presented in Fig. 1.

![ECB encryption](/assets/ECB%20encryption.png)
![ECB Decryption](/assets/ECB%20Decryption.png)

We can see it in Fig. 1, the plaintext is divided into blocks as the length of the block of AES, 128.
So the ECB mode needs to pad data until it is same as the length of the block.
Then every block will be encrypted with the same key and same algorithm.
So if we encrypt the same plaintext, we will get the same ciphertext.
So there is a high risk in this mode. And the plaintext and ciphertext blocks are a one-to-one correspondence.
Because the encryption/ decryption is independent, so we can encrypt/decrypt the data in parallel.
And if a block of plaintext or ciphertext is broken, it won't affect other blocks.

Because of the feature of ECB, the Mallory can make an attack even if they don't get the plaintext.
For example, if we encrypt the data about our bank account, like this:
The ciphertext:
C1: 21 33 4e 5a 35 44 90 4b(the account)
C2: 67 78 45 22 aa cb d1 e5(the password)
Then the Mallory can copy the data in C1 to C2. Then he can log in the system with the account as the password which is easier to get.

In the database encryption, we can use ECB to encrypt the tables, indexes, wal, temp files, and system catalogs.
But with the issues of security, we don't suggest to use this mode.

### CBC mode
The CBC (Cipher Block Chaining) mode (Fig. 2) provides this by using an initialization vector – IV.
The IV has the same size as the block that is encrypted. In general, the IV usually is a random number, not a nonce.

![CBC encryption](/assets/CBC%20encryption.png)
![CBC Decryption](/assets/CBC%20Decryption.png)

We can see it in figure 2, the plaintext is divided into blocks and needs to add padding data.
First, we will use the plaintext block xor with the IV. Then CBC will encrypt the result to the ciphertext block.
In the next block, we will use the encryption result to xor with plaintext block until the last block.
In this mode, even if we encrypt the same plaintext block, we will get a different ciphertext block.
We can decrypt the data in parallel, but it is not possible when encrypting data.
If a plaintext or ciphertext block is broken, it will affect all following block.

A Mallory can change the IV to attack the system. Even if a bit is wrong in the IV, all data is broken.
Mallory also can make a padding oracle attack. They can use a part of ciphertext to pad the ciphertext block.
This will return some messages about the plaintext.
It is safe from CPA, but it is easily sysceptible to CCA and PA.

To ensure security, we need to change the key when we encrypt 2^((n+1)/2)(n is the length of a block).

### CFB mode
The CFB (Cipher FeedBack) mode of operation allows the block encryptor to be used as a stream cipher. It also needs an IV.

![CFB encryption](/assets/CFB%20encryption.png)
![CFB Decryption](/assets/CFB%20Decryption.png)

First, CFB will encrypt the IV, then it will xor with plaintext block to get ciphertext. Then we will encrypt the encryption result to xor the plaintext.
Because this mode will not encrypt plaintext directly, it just uses the ciphertext to xor with the plaintext to get the ciphertext. So in this mode, it doesn't need to pad data.

And it could decrypt data in parallel, not encryption.
This mode is similar to the CBC, so if there is a broken block, it will affect all following block.

This mode can be attacked by replay attack. For example, if you use the other ciphertext to replace the new ciphertext, the user will get the wrong data.
But he will not know the data is wrong.
It is safe from CPA, but it is easily sysceptible to CCA.

To ensure security, the key in this mode need to be changed for every  2^((n+1)/2) encryption blocks.

### OFB mode
The OFB (Output FeedBack) mode of operation (Fig. 4) also enables a block encryptor to be used as a stream encryptor.
It also doesn't need padding data.

![OFB encryption](/assets/OFB%20encryption.png)
![OFB Decryption](/assets/OFB%20Decryption.png)

In this mode, it will encrypt the IV in the first time and encrypt the per-result.  Then it will use the encryption results to xor the plaintext to get ciphertext.
It is different from CFB, it always encrypts the IV. It can not encrypt/decrypt the IV in parallel.
Please note that we won't decrypt the IV encryption results to decrypt data.
It will not be affected by the broken block.
It is safe from CPA, but it is easily sysceptible to CCA and PA.

A Mallory can change some bits of ciphertext to damage the plaintext.

To ensure security, the key in this mode need to be changed for every  2^(n/2) encryption blocks.

### CTR mode
At the CTR (Counter) mode of operation, shown in Fig. 5, as an input block to the encryptor (Encrypt), i.e. as an IV, the value of a counter (Counter, Counter + 1,…, Counter + N - 1) is used. It also is a stream encryptor.

![CTR encryption](/assets/CTR%20encryption.png)
![CTR Decryption](/assets/CTR%20Decryption.png)

The counter has the same size as the used block. As shown in Fig. 5, the XOR operation with the block of plain text is performed on the output block from the encryptor. All encryption blocks use the same encryption key.
As this mode,  It will not be affected by the broken block.
It is very like OFB. But CTR will use the counter to be encrypted every time instead of the IV.
So if you could get counter directly, you can encrypt/decrypt data in parallel.

A Mallory can change some bits of ciphertext to break the plaintext.
In the database encryption, we can use CBC to encrypt all the files.

To ensure security, the key in this mode need to be changed for every  2^(n/2) encryption blocks.

Summary

![diff mode](/assets/diff%20mode_sgplwzari.png)

Which is the database needs?
| Object name     | alignment | Parallel write | Parallel read | Encryption Mode    |
| --------------- | --------- | -------------- | ------------- | ------------------ |
| Indexes         | yes       | yes            | yes           | CTR                |
| WAL             | no        | no             | no            | CFB, OFB, CTR      |
| System catalogs | no        | no             | yes           | CBC, CTR, CFB      |
| Temporary files | yes       | no             | no            | CBC, CTR, CFB, OFB |
| Tables | yes | no | yes | CBC, CTR, CFB |

In the end, I think the CTR mode is the best mode for PostgreSQL.
