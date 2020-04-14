---
layout: post
title:  "PKI adventures - part 1"
date:   2017-12-23 13:00:00 -0500
categories: [pki]
tags: [pki, kubernetes]
toc: true
---
This will be a post about my adventures to create a proper PKI environment. 

To bootstrap knowledge on the different terms used in here, I suggest to have a look at [this page](https://www.digitalocean.com/community/tutorials/a-comparison-of-let-s-encrypt-commercial-and-private-certificate-authorities-and-self-signed-ssl-certificates).

Planning phase
========================

Before deep-diving into that rabbit hole, I've tried to lay down some requirements. First thing that came to mind was the ability to get these certificates automatically via an API and get them renewed. Thinking about it, I was starting to describe what [Let's encrypt](https://letsencrypt.org/) does really well. At that point, I've put it on a short list before continuing with my list of requirements.

```
List of requirements:
========================
- Certificates must be created automatically via an API.
```

I've then started to think about where I'd use these certificates and then I've started to think about securing my Kubernetes clusters. While looking at what I could do with the Kubernetes clusters and it's PKI implementation (see [here](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/), [here](https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/) and [here](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)) the list of requirements suddenly got longer.

```
List of requirements:
========================
- Certificates must be created automatically via an API.
- Must be able to create a private certificate authority
- Must be able to issue server and client certificates.
- Must be able to issue wildcard certificates.
- Must be able to issue certificates with private IP addresses.
```

I next thought about securing [Docker](https://docs.docker.com/engine/security/https/) and the requirements list held still.

Thinking I had a pretty complete list, I started looking at already available tools/services to do this and looked at my first candidate from the short list, Let's encrypt. I've quickly removed it from the list because it must generate certificates for domains that are publicly accessible and cannot generate wildcard certificates *(available January 2018)* and certificates for private IP addresses.

A bit bummed out, this is when I entered the rabbit hole. I've started to look at how it worked and if there was any other tool working that way I could leverage and I started to learn about the [ACME proposed standard](https://ietf-wg-acme.github.io/acme/draft-ietf-acme-acme.html).

Going further down the rabbit hole, I started looking for tools that implemented this and I landed on a [blog from CloudFlare](https://blog.cloudflare.com/introducing-cfssl/) which was pretty interesting. That gave me the idea of adding the requirement to my list to be able to create a root and several intermediate certificate authorities, basically one per "computing environment" I had. 

```
List of requirements:
========================
- Certificates must be created automatically via an API.
- Must be able to create a private certificate authority
- Must be able to create root, intermetiate certificate authorities
- Must be able to issue server and client certificates.
- Must be able to issue wildcard certificates.
- Must be able to issue certificates with private IP addresses.
```

Doing further reading on the web about how normal certificate authorities work, I've decided to adopt the following layout:

![PKI diagram]({{ "/assets/images/PKI-diagram.png" | absolute_url }})

Because of the way the PKI infrastructure works, based on trust, I think this layout has the following advantages:

- Lessens the amount of certificates to put in the various trust stores for our internal PKI infrastructure to be automatically accepted by our systems.
- Mitigates the risks of having a CA compromised: we can put extra security on the root and intermediate CA while having semi-automated/automated means of generating certificates per environment.

Testing phase
===============

To see if it was realisting and feasable to implement, I starting playing with the CFSSL toolkit to create the various CAs. I searched a bit on the internet and found the following links on how to create CAs with CFSSL:

- [https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/](https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/)
- [https://coreos.com/os/docs/latest/generate-self-signed-certificates.html](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)
- [https://github.com/jason-riddle/generating-certs/wiki/Generating-a-Root-CA,-Server,-and-Client-Certs-using-CFSSL](https://github.com/jason-riddle/generating-certs/wiki/Generating-a-Root-CA,-Server,-and-Client-Certs-using-CFSSL)
- [https://technedigitale.com/archives/639](https://technedigitale.com/archives/639)
- [https://github.com/kelseyhightower/docker-kubernetes-tls-guide](https://github.com/kelseyhightower/docker-kubernetes-tls-guide)

While reading all of these, I've found that CFSSL has changed a bit the way to define the CA configuration depending on the version of the binaries. At the time of writing this, the version you can download from the CFSSL github site is not the same as the one you're going to get using `brew` on a Mac. This detail get can you a few hours of headaches. I strongly suggest to download the latest version of the binaries from the site or to compile yours using Go from the sources instead of using an operating system package.

CFSSL config
----------------------

- CFSSL configuration file (*config.json*):
```json
{
   "signing":{
      "default":{
         "expiry":"43800h"
      },
      "profiles":{
         "root":{
            "expiry":"262800h",
            "ca_constraint":{
               "is_ca":true,
               "max_path_len":2,
               "max_path_len_zero":false
            },
            "usages":[
               "digital signature",
               "cert sign",
               "crl sign",
               "signing"
            ]
         },
         "intermediate":{
            "expiry":"131400h",
            "ca_constraint":{
               "is_ca":true,
               "max_path_len":1,
               "max_path_len_zero":false
            },
            "usages":[
               "digital signature",
               "cert sign",
               "crl sign",
               "signing"
            ]
         },
         "leaf":{
            "expiry":"87600h",
            "ca_constraint":{
               "is_ca":true,
               "max_path_len":0,
               "max_path_len_zero":true
            },
            "usages":[
               "digital signature",
               "cert sign",
               "crl sign",
               "signing"
            ]
         },
         "server":{
            "expiry":"43800h",
            "usages":[
               "signing",
               "key encipherment",
               "server auth"
            ]
         },
         "client":{
            "expiry":"43800h",
            "usages":[
               "signing",
               "key encipherment",
               "client auth",
               "digital signature"
            ]
         },
         "client-server":{
            "expiry":"43800h",
            "usages":[
               "signing",
               "key encipherment",
               "server auth",
               "client auth",
               "digital signature"
            ]
         }
      }
   }
}
```


CFSSL CSRs
----------------------

- Root CA CSR (*root.json*):
```json
{
   "CN":"My Root CA",
   "key":{
      "algo":"ecdsa",
      "size":256
   },
   "names":[
      {
         "C":"CA",
         "ST":"Quebec",
         "L":"Montreal",
         "O":"Fortkickass"
      }
   ],
   "ca":{
      "expiry":"262800h"
   }
}
```

- Intermediate CA CSR (*intermediate.json*):
```json
{
   "CN":"My Intermediate CA",
   "key":{
      "algo":"ecdsa",
      "size":256
   },
   "names":[
      {
         "C":"CA",
         "ST":"Quebec",
         "L":"Montreal",
         "O":"Fortkickass"
      }
   ],
   "ca":{
      "expiry":"131400h"
   }
}
```

- Leaf CA CSR (*leaf.json*):
```json
{
   "CN":"My Leaf CA",
   "key":{
      "algo":"ecdsa",
      "size":256
   },
   "names":[
      {
         "C":"CA",
         "ST":"Quebec",
         "L":"Montreal",
         "O":"Fortkickass"
      }
   ],
   "ca":{
      "expiry":"87600h"
   }
}
```

- Server Certificate CSR (*server.json*):
```json
{
  "CN": "example.com",
  "hosts": [
    "127.0.0.1",
    "localhost",
    "*.example.com",
    "example.com"
  ],
  "key": {
    "algo": "ecdsa",
    "size": 256
  },
  "names": [
    {
         "C":"CA",
         "ST":"Quebec",
         "L":"Montreal",
         "O":"Fortkickass"
    }
  ]
}
```

Certificate generation
----------------------

1. Create the root CA.
```shell
$ cfssl gencert -initca root.json | cfssljson -bare root
2017/12/23 16:25:45 [INFO] generating a new CA key and certificate from CSR
2017/12/23 16:25:45 [INFO] generate received request
2017/12/23 16:25:45 [INFO] received CSR
2017/12/23 16:25:45 [INFO] generating key: ecdsa-256
2017/12/23 16:25:45 [INFO] encoded CSR
2017/12/23 16:25:45 [INFO] signed certificate with serial number 645706087224409591316507428217837704988068866053
$ openssl x509 -noout -text -in root.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            71:1a:76:5c:8f:44:ae:c6:64:10:bf:d9:a3:d2:d9:68:a2:ae:e8:05
    Signature Algorithm: ecdsa-with-SHA256
        Issuer: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=My Root CA
        Validity
            Not Before: Dec 23 21:21:00 2017 GMT
            Not After : Dec 16 21:21:00 2047 GMT
        Subject: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=My Root CA
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:19:2a:38:7f:52:ba:f8:01:04:79:67:c5:dc:97:
                    ec:22:b1:37:8c:99:d7:0a:3c:dd:0c:22:73:e5:d3:
                    0b:b5:9f:5c:96:f9:53:b6:43:14:5f:23:14:ac:01:
                    67:8e:8c:7a:93:3a:e8:5e:e7:7d:db:bc:ff:54:d7:
                    d1:f5:06:0e:ea
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                F9:E9:6F:33:26:DA:51:34:12:0D:9C:27:7B:7F:7C:5A:E3:23:65:DB
    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:21:00:a7:b7:aa:fa:65:e4:0d:0a:e0:1e:c6:de:47:
         d6:a1:7e:03:7b:5f:08:b7:82:e2:d5:f2:9d:7c:65:af:15:df:
         da:02:20:42:8a:63:c4:24:9a:8c:c1:61:cb:91:7a:ed:2d:ec:
         34:46:fa:26:86:4c:b0:e7:f4:86:b6:f3:c0:29:3a:95:7a
```

2. Create the intermediate CA.
```shell
$ cfssl gencert -initca intermediate.json | cfssljson -bare intermediate
2017/12/23 16:28:50 [INFO] generating a new CA key and certificate from CSR
2017/12/23 16:28:50 [INFO] generate received request
2017/12/23 16:28:50 [INFO] received CSR
2017/12/23 16:28:50 [INFO] generating key: ecdsa-256
2017/12/23 16:28:50 [INFO] encoded CSR
2017/12/23 16:28:50 [INFO] signed certificate with serial number 550155926647042054619769394624297047559842575481
$ cfssl sign -ca root.pem -ca-key root-key.pem -config config.json -profile intermediate intermediate.csr | cfssljson -bare intermediate
2017/12/23 16:31:31 [INFO] signed certificate with serial number 685400036135247529612056362867615885254114452526
$ openssl x509 -noout -text -in intermediate.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            78:0e:66:8c:53:70:ac:00:63:84:58:35:5c:df:d5:56:ef:d0:a4:2e
    Signature Algorithm: ecdsa-with-SHA256
        Issuer: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=My Root CA
        Validity
            Not Before: Dec 23 21:27:00 2017 GMT
            Not After : Dec 19 21:27:00 2032 GMT
        Subject: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=My Intermediate CA
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:f3:c8:7b:1a:ba:e2:d7:57:ef:94:71:c4:d7:be:
                    7a:9c:fa:cc:1f:de:6b:2f:d3:39:7d:34:6b:5d:a8:
                    de:a4:7e:41:da:f3:bd:2b:f3:b1:01:8a:c2:39:a3:
                    03:6a:54:d2:da:46:26:bb:c7:c4:e5:cb:91:fb:7b:
                    d8:12:bb:b5:e0
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:1
            X509v3 Subject Key Identifier:
                15:A8:12:C6:9E:7F:38:23:1A:DF:1B:C1:44:8B:BA:11:8F:7D:CC:72
            X509v3 Authority Key Identifier:
                keyid:F9:E9:6F:33:26:DA:51:34:12:0D:9C:27:7B:7F:7C:5A:E3:23:65:DB

    Signature Algorithm: ecdsa-with-SHA256
         30:44:02:20:23:30:7b:59:62:04:42:af:49:c5:f1:12:30:b5:
         31:29:0d:bf:7f:50:a4:e2:a5:bc:c9:26:f0:bd:61:59:29:7b:
         02:20:67:28:9c:f4:57:90:a9:a6:60:74:c4:83:64:0f:9e:69:
         94:46:8d:6b:c2:4d:13:79:e9:cc:f4:17:b2:70:3e:6b
```

3. Encrypt the root certificate private key.
```shell
$ mv root-key.pem root-key-cleartext.pem
$ openssl ec -in root-key-cleartext.pem -out root-key.pem -aes256
read EC key
writing EC key
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
$ rm -f root-key-cleartext.pem
$ chmod 600 root-key.pem
```

4. Create the leaf CA.
```shell
$ cfssl gencert -initca leaf.json | cfssljson -bare leaf
2017/12/23 16:33:29 [INFO] generating a new CA key and certificate from CSR
2017/12/23 16:33:29 [INFO] generate received request
2017/12/23 16:33:29 [INFO] received CSR
2017/12/23 16:33:29 [INFO] generating key: ecdsa-256
2017/12/23 16:33:29 [INFO] encoded CSR
2017/12/23 16:33:29 [INFO] signed certificate with serial number 224002282509320265206540380849484135592958791791
$ cfssl sign -ca intermediate.pem -ca-key intermediate-key.pem -config config.json -profile leaf leaf.csr | cfssljson -bare leaf
2017/12/23 16:34:12 [INFO] signed certificate with serial number 474590541869854391415994307532174409966075318689
$ openssl x509 -noout -text -in leaf.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            53:21:60:3c:4d:e3:ad:fe:ba:e9:39:de:ce:f9:d1:ed:78:e5:11:a1
    Signature Algorithm: ecdsa-with-SHA256
        Issuer: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=My Intermediate CA
        Validity
            Not Before: Dec 23 21:29:00 2017 GMT
            Not After : Dec 21 21:29:00 2027 GMT
        Subject: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=My Leaf CA
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:6a:1f:3e:29:2d:cf:be:3e:14:87:02:5d:84:10:
                    f0:f3:bc:25:02:24:fa:2d:f1:0e:9a:f2:e6:46:d2:
                    b1:41:30:79:cd:e9:f4:b1:f0:ec:17:7b:27:7f:3b:
                    d1:96:94:4d:73:43:e9:27:d1:b1:66:69:da:f8:84:
                    fc:23:db:d5:ae
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Subject Key Identifier:
                0E:16:70:91:BE:75:10:E6:95:D7:84:34:6A:E5:B7:39:8E:B0:7E:2A
            X509v3 Authority Key Identifier:
                keyid:15:A8:12:C6:9E:7F:38:23:1A:DF:1B:C1:44:8B:BA:11:8F:7D:CC:72

    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:20:76:cf:f7:1d:7e:b6:41:77:41:77:5a:4c:34:fa:
         b8:1d:5e:cf:7c:8b:79:b8:11:f0:5d:62:0f:6d:c7:75:75:0f:
         02:21:00:a4:57:16:e6:22:b9:d0:8b:62:a0:fc:ef:ab:7c:32:
         1f:c5:1d:39:b9:67:84:05:79:2e:83:da:31:5d:4f:7a:fe
$ chmod 600 leaf-key.pem
```

5. Encrypt the intermediate certificate private key.
```shell
$ mv intermediate-key.pem intermediate-key-cleartext.pem
$ openssl ec -in intermediate-key-cleartext.pem -out intermediate-key.pem -aes256
read EC key
writing EC key
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
$ rm -f intermediate-key-cleartext.pem
$ chmod 600 intermediate-key.pem
```

6. Create the certificate chain bundle.
```shell
$ cat leaf.pem intermediate.pem root.pem > ca-chain.pem
```

7. Create the server certificate.
```shell
$ cfssl gencert -ca leaf.pem -ca-key leaf-key.pem -config config.json -profile server server.json | cfssljson -bare server
2017/12/23 16:45:02 [INFO] generate received request
2017/12/23 16:45:02 [INFO] received CSR
2017/12/23 16:45:02 [INFO] generating key: ecdsa-256
2017/12/23 16:45:02 [INFO] encoded CSR
2017/12/23 16:45:02 [INFO] signed certificate with serial number 372589075770299352005490224527061857910716158361
$ chmod 600 server-key.pem
$ openssl x509 -noout -text -in server.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            41:43:78:d4:5d:7e:6c:74:1b:38:99:72:8d:35:fc:d6:3f:a4:d1:99
    Signature Algorithm: ecdsa-with-SHA256
        Issuer: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=My Leaf CA
        Validity
            Not Before: Dec 23 21:40:00 2017 GMT
            Not After : Dec 22 21:40:00 2022 GMT
        Subject: C=CA, ST=Quebec, L=Montreal, O=Fortkickass, CN=example.com
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:57:4d:15:a6:58:47:0e:2f:3b:70:84:63:2a:d4:
                    5a:fb:3f:62:47:b6:57:73:7f:2e:60:9b:a3:0b:d7:
                    4f:67:c4:50:a1:71:c5:d8:cb:e3:a0:57:ce:a8:45:
                    a2:03:0c:3e:39:8f:89:6b:ed:d0:56:2a:b5:30:a0:
                    3e:73:6d:5c:af
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                0B:A8:D9:D9:70:97:09:F8:2D:03:61:00:B5:EE:BF:2C:B9:90:9A:FB
            X509v3 Authority Key Identifier:
                keyid:0E:16:70:91:BE:75:10:E6:95:D7:84:34:6A:E5:B7:39:8E:B0:7E:2A

            X509v3 Subject Alternative Name:
                DNS:localhost, DNS:*.example.com, DNS:example.com, IP Address:127.0.0.1
    Signature Algorithm: ecdsa-with-SHA256
         30:46:02:21:00:b4:6e:b1:4a:c9:09:2b:fd:1d:80:76:9a:b0:
         e2:a2:b8:f7:07:72:c7:22:49:c8:df:ee:14:06:6d:a1:d8:d6:
         10:02:21:00:90:a2:5a:8b:ac:e9:7c:40:a9:ee:ea:61:42:9d:
         6c:80:b3:be:69:92:19:83:e4:8e:02:76:7a:9c:d4:3f:b0:7b
```

8. Verify the certificate.
```shell
$ openssl verify -CAfile ca-chain.pem server.pem
server.pem: OK
```

Since we now know we can generate valid certificates with our leaf CA, we can now focus on finding the right solution to automate certificate creation.
