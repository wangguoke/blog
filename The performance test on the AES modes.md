After introducing the difference between the AES modes, in this document, I will put the results about the AES modes performance.

The following tests just use one core CPU.

AES-NI:
>The Advanced Encryption Standard Instruction Set (or Intel Advanced Encryption Standard New Instructions, AES-NI for short) is an extension of the x86 instruction set architecture for Intel and AMD microprocessors, presented by Intel in March 2008. [1] The purpose of this instruction set is to improve the speed at which applications use the Advanced Encryption Standard (AES) to perform encryption and decryption.

OpenSSL:
>OpenSSL is a robust, commercial-grade, and full-featured toolkit for the Transport Layer Security (TLS) and Secure Sockets Layer (SSL) protocols. It is also a general-purpose cryptography library. There is much Standard encryption algorithm in OpenSSL. We will use OpenSSL to test the AES modes performance.*

### How to get the performance results
You can refer to the official document:
https://www.openssl.org/docs/manmaster/man1/speed.html.

Here we will use the following command to do the performance test.
With AES-NI enabled
```
openssl speed -elapsed -evp aes-128-cbc
```
With disabled AES-NI
```
OPENSSL_ia32cap="~0x200000200000000" openssl speed -elapsed -evp aes-128-cbc
```
Performance test server configuration:
* CPU : i5 8400 (has the AES-NI)
* Memory : 16G DDR4
* Disk : Inter SSD 1T
* OS : CentOS Linux release 7.6.1810 (Core)
* OpenSSL :  OpenSSL 1.0.2k

The tests for each input data size was performed for 3 seconds, for the ciphers that we were interested in.

Five modes with 128-bits key, AES-NI enabled and disabled, encryption
(the first row means OpenSSL will use ase-ecb with 128-bits key to encrypted 1371968.28k data in 3 seconds):
![5 modes speed](/assets/5%20modes%20speed.png)
In the result, we can get the ECB is the fastest mode, but it is not be recommended, we suggest to use the CTR mode in the PostgreSQL to encrypt.
After 1024 bytes, the speed is nearly the same, so we suggest to use the 8192 bytes as a unit to encrypt in the PostgreSQL.
At the same time, we can know that AES-NI will have up to 10 times the performance gap between opening and closing.

After comparing the different modes in the horizontal direction, we will perform performance tests with different key lengths in the same mode.

As the above test knows, we will use ctr mode encryption, so here we only test the performance comparison of different key lengths of ctr mode.
CTR mode:
![3 sizes in ctr mode](/assets/3%20sizes%20in%20ctr%20mode.png)
In the results, we can know that as the key length increases, the encryption speed will also decrease.
However, it is well known that as keys grow, security increases.
How to balance the relationship between the two will be investigated later.
But we can know that if you think speed is more important than security,
you can use a 128-bit key, otherwise, you can use a 256-bit key.

Five modes with 128-bits key, AES-NI enabled, encryption and decryption:
![5 modes ende](/assets/5%20modes%20ende.png)
Except for cbc mode, the encryption and decryption speed of all modes is almost the same.

In the end, comparing the encryption and decryption speeds of different modes, the encryption speed of different block sizes,
the encryption speed of different key lengths, and the encryption speed of turning AES-NI on and off,
I recommend using CTR mode for data encryption in PostgreSQL.
