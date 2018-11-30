# k8s-setup-docs
This is repo provides the documentation steps followed to setup k8s in aws environment 

# Getting Started

Welcome to the world of k8s!. You might be wondering about how all the things work so perfectly in k8s.
This setup will walk you through in installing each and every component from the SCRATCH!
Yes you heard me right !
And the emphasis is on production environment.

> *Note*
>We are using the AWS Cloud for the whole process. It's isn't native to AWS we can run on any cloud.
>It was the option for us.

In order to achieve this we need to have :
1. 3 VM's in EC2 (works on any cloud) able to talk to one another ( 1 master, 1 worker , 1 etcd server)
2. Peace of mind,patience in abundance !

## Hardware Requirements:
1. Master( contains apiserver, scheduler, controller etc.) - The Kubernetes control pane T2.Medium
2. Worker - T2.Medium
3. Etcd server - Data store for all the components of k8s

## Utilities needed:
1.  cfssl: It's used to generate openssl certs for the kubernetes components.

```
    wget -q --show-progress --https-only --timestamping \
    https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
    https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
```

``` chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 
    sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl 
    sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

2. kubectl: We need it to talk to the k8s cluster

```
    wget https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl
    chmod +x kubectl
    sudo mv kubectl /usr/local/bin/
```
## SSL Certificates Generation
### CA Authority

``` 
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF
```

``` 
cfssl gencert -initca ca-csr.json | cfssljson -bare ca 
```
The above command will generate certificates for CA using csr mentioned above.

### Client and Server Certificates
This section will generate certificates for k8s components and a client cerficate for k8s admin user

#### The Admin Client Certificate
```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin 
```
The above command will generate the admin certificate 

#### Kubelet Certificate Generation
Kubernetes uses a special-purpose authorization mode called Node Authorizer, that specifically authorizes API requests made by Kubelets.
Kubelets must use a credential that identifies them as being in the 'system:nodes' group, with a username of system:node:<nodeName>
Generate the certificates using the below steps to add your worker node as authorized to perform API requests

```
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

``` EXTERNAL_IP=<Public IP of the Machine> ```

``` INTERNAL_IP=<Private IP of the Machine> ```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=<Worker>,${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  <Worker>-csr.json | cfssljson -bare <Worker>
  ```
> Note:
> As I have only one worker node there is only one set of certificates.
 
 #### Controller Manager Certificate

```
 cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
 ```

#### Kubeproxy Client Certificate

``` 
 cat > kube-proxy-csr.json <<EOF
 {
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```
#### Scheduler Client Certificate
```
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

#### Kubernetes API Server Certificate

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

```
cfssl gencert \
   -ca=ca.pem \
   -ca-key=ca-key.pem \
   -config=ca-config.json \
   -hostname=<List of IP's in the K8s cluster>,<API Server Public IP>,127.0.0.1,kubernetes.default \
   -profile=kubernetes \
    kubernetes-csr.json | cfssljson -bare kubernetes
```

The above steps needs to done carefully as the nodes will fail to communicate if the IP is missed in list.
We can't talk with k8s apiserver

#### Service Account Key Pair
The Kuberenetes Controller Manager leverarges a key pair to generate and sign service account tokens.

 ```
 cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

```
 cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

Now the crucial part distributing these certificates to the machines

| Client Certificates       | Server Certificates        |
| ------------------------ |:-------------------------:|
| Admin Certificate   | Kuberenetes API Server |
| Kubelet Client Certificate   |
| Controller Manager Client Certificate   | 
| Kube Proxy Client Certificate   |
| Scheduler Client Certificate    |

Except Admin we will use all the client certificates to generate client authentication configuration files

#### Copying the certs to Worker Nodes

These are the certificates should be copied into worker nodes

* `ca.pem`
* `<worker>-key.pem`
* `<worker>.pem`

#### Copying the certs Controller Nodes

These are the certificates should be present in the Controller Nodes

* `ca.pem`
* `ca-key.pem`
* `kubernetes-key.pem`
* `kubernetes.pem`
* `service-account-key.pem` 
* `service-account.pem`
