# Stux CTF writeup

## Enumeration

Using `sudo nmap -Pn -p- -v -A 10.10.189.199` to find open ports, and which services reside at these ports.

![nmap scan](images/nmap.png)

The scan found an endpoint (`/StuxCTF/`) in the robots.txt folder. This page, however, gives us a 404 exception.


Heading back to the index of port 80 I inspect the element and find something about a secret directory.

![secret directory](images/inspect.png)

This problem immediately made me think of a discrete logarithm problem. More spesifically Diffie Hellman.
p is a public prime, and g a public generator, and a and b are Alice's and Bob's respective private keys. g^c might be a third person's g^c mod p public key.

In order to create a shared secret key using Diffie Hellman, you simply combine all peoples public keys together with your own private key like this:

shared_secret = (a_public * b_public)^(c_private) mod p = (g^a_private mod p * g^b_private mod p)^(c_private) mod p = g^(a_private*b_private*c_private) mod p

![dh](images/dh.png)
![dh short](images/128dh.png)

Aaaand, that's our first flag!

1. What is the hidden directory?
    * 47315028937264895539131328176684350732577039984023005189203993885687328953804202704977050807800832928198526567069446044422855055