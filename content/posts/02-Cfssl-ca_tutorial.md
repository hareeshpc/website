---
title: "Using CFSSL for CA and Certs"
date: 2020-10-26T22:21:01+01:00
draft: false
---

This is a working example of creating a CA , intermediate CA and leaf certificates based on CFSSL

Note: These notes are based on cfssl version 1.4.1

> This tutorial is based on the blog entry by Johannes [ca-with-cfssl](https://jite.eu/2019/2/6/ca-with-cfssl/). It has been adapted, with minor improvements.Fixes were made for working with cfssl 1.4.1. 

## Pre-requisites

- Obtain `cfssl` and `cfssljson` binaries and put them in the path.

- Create a directory heirarchy as follows.

```
/pki
  /root-ca
  /intermediate-ca
    /dev
    /prod
  /leaf-certs 
```

```bash
mkdir -p pki/{root-ca,intermediate-ca,leaf-certs}
mkdir -p pki/intermediate-ca/{dev,prod}
```

- Set Env variables for ease

```bash 
ROOT=$(pwd)/pki
ROOT_CA_PATH=${ROOT}/root-ca
INTERMEDIATE_CA_PATH=${ROOT}/intermediate-ca
DEV_INTERMED_CA_PATH=${ROOT}/intermediate-ca/dev
PROD_INTERMED_CA_PATH=${ROOT}/intermediate-ca/prod
LEAF_CERTS_PATH=${ROOT}/leaf-certs
```


## Root CA

### Root CA SR

> Note: All comamnds are run inside the `root-ca` folder
```bash
cat <<EOF >> root-ca-sr.json
{
    "CN": "Hareesh Root CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
               "C": "SE",
               "L": "Stockholm",
               "O": "Company",
               "ST": "Stockholm"
        }
    ]
}
EOF
```
### Legend
The names array contains an object, the short codes means: 
- C ISO 3166-2 country code (SE = Sweden), 
- L Locality, I.E., city name of the company, 
- O Organization, 
- ST State. 

You could also add OU, which stands for Organizational Unit, to this list if you wish.
In this case, it is not needed as this is the root of all the certs.

### Root certs

```bash
cfssl gencert -initca ${ROOT_CA_PATH}/root-ca-sr.json | cfssljson -bare ca
```

## Intermediate CA  

All the intermediate CA have an `expiry` time of 43800 hours or ~ 5 years. 

> Note: All commands are run inside the `intermediate-ca` folder

Create `/intermediate-ca/config.json`

### Intermediate SR and config
```bash
cat <<EOF > ${INTERMEDIATE_CA_PATH}/config.json
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "dev": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "cert sign",
                    "crl sign",
                    "server auth",
                    "client auth"
                ],
		        "expiry": "43800h",
                "ca_constraint": {
                    "is_ca": true
                }
            },
            "prod": {
                "usages": [     
                    "signing",
                    "key encipherment",
                    "cert sign",
                    "crl sign"
                ],
		        "expiry": "43800h",
                "ca_constraint": {
                    "is_ca": true
                }
            }
        }
    }
}
EOF

cat <<EOF > ${INTERMEDIATE_CA_PATH}/intermediate-ca.json
{
    "CN": "Hareesh Intermed CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
               "C": "SE",
               "L": "Stockholm",
               "O": "Company",
               "ST": "Stockholm"
        }
    ]
}
EOF
```
### Intermediate CA certs


#### Dev Intermediate CA

> For dev : All commands are run inside `intermediate-ca/dev` folder
```bash
cd ${DEV_INTERMED_CA_PATH}
cfssl genkey -initca ${INTERMEDIATE_CA_PATH}/intermediate-ca.json | cfssljson -bare dev

cfssl sign -ca ${ROOT_CA_PATH}/ca.pem -ca-key ${ROOT_CA_PATH}/ca-key.pem --config ${INTERMEDIATE_CA_PATH}/config.json -profile dev  dev.csr | cfssljson -bare dev
```

#### Prod Intermediate CA
> For prod : All commands are run inside `intermediate-ca/prod` folder
```bash
cd ${PROD_INTERMED_CA_PATH}
cfssl genkey -initca ${INTERMEDIATE_CA_PATH}/intermediate-ca.json | cfssljson -bare prod

cfssl sign -ca ${ROOT_CA_PATH}/ca.pem -ca-key ${ROOT_CA_PATH}/ca-key.pem --config ${INTERMEDIATE_CA_PATH}/config.json -profile prod  prod.csr | cfssljson -bare prod
```

You will end up with a tree heirarchy like this
```
intermediate-ca
├── config.json
├── dev
│   ├── dev.csr
│   ├── dev-key.pem
│   └── dev.pem
├── intermediate-ca.json
└── prod
    ├── prod.csr
    ├── prod-key.pem
    └── prod.pem

2 directories, 8 files
```


## Leaf Certificates


Three kinds of certificates
 * server : For server based services expecting connections over TLS. 
 * client:  For clients who need to authenticate with servers
 * peer : Services who need to both authenticate to other services as well identify themselves via TLS.

The main difference between these three are  in the `usages` part as defined in [CFSSL Documentation Profiles](https://github.com/cloudflare/cfssl/blob/master/doc/cmd/cfssl.txt) (Go to Signing Profiles
 -> Key Usages) 

 * server: "signing","digital signing", "key encipherment",      "server auth"
 * client: "signing","digital signature","key encipherment",    "client auth"
 * peer: "signing", "digital signature", "key encipherment", "client auth", "server auth"


 ### Leaf Signing Usage Config

> Note: All commands are run inside the `leaf-certs` folder

```bash
cat <<EOF > ${LEAF_CERTS_PATH}/leaf-config.json
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "usages": [
                    "signing",
                    "digital signing",
                    "key encipherment",
                    "server auth"
                ],
            "expiry": "43800h"
            },
            "peer": {
                "usages": [
                    "signing",
                    "digital signature",
                    "key encipherment", 
                    "client auth",
                    "server auth"
                ],
            "expiry": "43800h"
            },
            "client": {
                "usages": [
                    "signing",
                    "digital signature",
                    "key encipherment", 
                    "client auth"
                ],
            "expiry": "43800h"
            }
        }
    }
}
EOF
```
### Signing profiles

* Server signing profile

```bash 
cat <<EOF > ${LEAF_CERTS_PATH}/server.json
{
    "CN": "My Service",
    "hosts": [
        "127.0.0.1",
        "mysvc.example.org",
        "mysvc-altname.example.org"
    ]
}
EOF
```

* Client Signing profile

```bash
cat <<EOF > ${LEAF_CERTS_PATH}/client.json
{
    "CN": "My User",
    "hosts": [""]
}
EOF
```

* Peer signing profile (optional)

```bash
cat <<EOF > ${LEAF_CERTS_PATH}/peer.json
{
    "CN": "Peer",
    "hosts": [
        "peersvc.example.org",
        ]
}
```

### Create Leaf certificates (Using Dev intermediate CA)

> Note: All commands are run inside the `leaf-certs` folder

* For server

```bash
cfssl gencert -ca=${DEV_INTERMED_CA_PATH}/dev.pem -ca-key=${DEV_INTERMED_CA_PATH}/dev-key.pem -config=leaf-config.json -profile=client client.json | cfssljson -bare client
```

* For client
```bash
cfssl gencert -ca=${DEV_INTERMED_CA_PATH}/dev.pem -ca-key=${DEV_INTERMED_CA_PATH}/dev-key.pem -config=leaf-config.json -profile=server server.json | cfssljson -bare server
```

* For peer (optional)
```bash
cfssl gencert -ca=${DEV_INTERMED_CA_PATH}/dev.pem -ca-key=${DEV_INTERMED_CA_PATH}/dev-key.pem -config=leaf-config.json -profile=peer peer.json | cfssljson -bare peer
```

After running the commands you should have the following in the `leaf-certs` folder.
```
leaf-certs/
├── client.csr
├── client.json
├── client-key.pem
├── client.pem
├── leaf-config.json
├── server.csr
├── server.json
├── server-key.pem
└── server.pem
```

### Verify certs

```bash
openssl x509 -in <cert>.pem -text -noout
```


## References
* [CFSSL documentation](https://github.com/cloudflare/cfssl/blob/master/doc/cmd/cfssl.txt)
* [CA with CFSSL](https://jite.eu/2019/2/6/ca-with-cfssl/)
