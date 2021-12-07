# Introduction

This demo presents a _privacy-preserving machine learning_ (PPML) application
use case built using the Gramine-SGX Library OS and PyTorch. It follows the
Gramine's [PyTorch PPML Framework Tutorial](https://gramine.readthedocs.io/en/latest/tutorials/pytorch/index.html)
but moves it to a Kubernetes orchestrated Cloud environment where the
application is run as a confidential container process inside an Intel
SGX enclave on an _untrusted machine_.

In the demo, both input data and the machine learning model are encrypted
inside the container and decrypted inside an Intel SGX based Trusted Execution
Environment (TEE) using a key delivered from a trusted host after a successfull
remote attestation. It shows how input data confidentiality as well as model's
IP can be guaranteed in an untrusted environment.

The high-level diagram (image credits: [Gramine project](https://gramine.readthedocs.io/)) is as follows (best viewed using Github's light theme):
![Gramine-SGX PyTorch PPML overview (image credits: the Gramine project)](https://raw.githubusercontent.com/gramineproject/gramine/master/Documentation/tutorials/pytorch/img/workflow.svg)

To simplify the setup, the _trusted machine_ and the _untrusted machine_ are run
as their own pods in a cluster (using either `runc` or `kata-runtime`). The setup
also assumes the cluster is SGX capable (step 1 in the diagram) and the data
encryption key is available (step 2).

# Flow

Upon the Gramine-SGX PyTorch startup, the secret provisioning library connects
to the trusted machine using a secure TLS (step 3) connection to get the data
encryption key. The TLS connection uses Gramine's remote attestation TLS (RA-TLS)
which allows the trusted machine to verify that the connection comes from a
genuine SGX enclave on the untrusted machine. If the remote attestation succeeds,
the data encryption key is released to the enclave.

Gramine's Protected FS transparently decrypts the encrypted input data and the
machine learning model as PyTorch runs (steps 6 and 7). Finally, the inference
result is encrypted and stored on a filesystem again. In this demo, the result
is copied on a hostPath mount for viewing.

# Setup

The setup is made as easy to use as possible. The base assumptions are:

* Kubernetes cluster with SGX enabled
* `cert-manager` is deployed
* Gramine is [installed](https://gramine.readthedocs.io/en/latest/devel/building.html) on the build host

## Generate `wrap-key`

```
$ pushd build
$ gramine-sgx-pf-crypt gen-key -w .
$ cp wrap-key ../deployments/secret-provider-server
```

## Build Gramine-SGX Pytorch image

```
$ docker build -t gramine:devel --force-rm .
$ popd
```

## Generate `cert-manager` CA Issuer key/cert

```
$ pushd deployments/cert-manager
$ openssl genrsa -out tls.key 2048
$ openssl req -x509 -new -nodes -key tls.key -subj "/CN=ccg-ca" -days 10000 -out tls.crt
$ popd
```

## Deploy

Make `gramine:devel` image available to your cluster and run

```
$ kubectl apply -k deployments/default
```

or with Kata Containers sandboxed Gramine

```
$ kubectl apply -k deployments/overlays/kata-containers
```

# Demo

Confidential Containers with Gramine-SGX and Pytorch
[<img src="https://asciinema.org/a/luYfWr3BTdYR7Ll0Xu2FcTyou.svg" width="700">](https://asciinema.org/a/luYfWr3BTdYR7Ll0Xu2FcTyou)

# References

* [Gramine project](https://gramine.readthedocs.io/en/latest/index.html)
* [PyTorch PPML Framework Tutorial](https://gramine.readthedocs.io/en/latest/tutorials/pytorch/index.html)
* [Kata Containers with Intel SGX](https://github.com/kata-containers/kata-containers/blob/main/docs/use-cases/using-Intel-SGX-and-kata.md)
* [Intel Device Plugins for Kubernetes - SGX](https://github.com/intel/intel-device-plugins-for-kubernetes#sgx-device-plugin)
