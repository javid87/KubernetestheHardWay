# KubernetestheHardWay
Kubernetes the Hard Way

- [Getting Started](#getting-started)
    - [What Will the Kubernetes Cluster Architecture Look Like?](#what-will-the-kubernetes-cluster-architecture-look-like)
    - [Setting Up Your Cloud Servers](#setting-up-your-cloud-servers)
    - [Client Tools](#client-tools)
- [Provisioning the CA and Generating TLS Certificates](#provisioning-the-ca-and-generating-tls-certificates)
    - [Why Do We Need a CA and TLS Certificates?](#why-do-we-need-a-ca-and-tls-certificates)
    - [Provisioning the Certificate Authority](#provisioning-the-certificate-authority)
    - [Generating Client Certificates](#generating-client-certificates)
    - [Generating the Kubernetes API Server Certificate](#generating-the-kubernetes-api-server-certificate)
    - [Generating the Service Account Key Pair](#generating-the-service-account-key-pair)
    - [Distributing the Certificate Files](#distributing-the-certificate-files)
    - [Creating a Certificate Authority and TLS Certificates for Kubernetes](#creating-a-certificate-authority-and-tls-certificates-for-kubernetes)
- [Generating Kubernetes Configuration Files for Authentication](#generating-kubernetes-configuration-files-for-authentication)
    - [What are Kubeconfigs and Why Do We Need Them?](#what-are-kubeconfigs-and-why-do-we-need-them)
    - [Generating Kubeconfigs for the Cluster](#generating-kubeconfigs-for-the-cluster)
    - [Distributing the Kubeconfig Files](#distributing-the-kubeconfig-files)
    - [Generating Kubeconfigs for a New Kubernetes Cluster](#generating-kubeconfigs-for-a-new-kubernetes-cluster)


## Getting Started 
### What Will the Kubernetes Cluster Architecture Look Like?
Structure going to be build during the course, according to the Kubernetes the Hard Way documentation. 
![img](https://github.com/Bes0n/KubernetestheHardWay/blob/master/images/img1.png)


### Setting Up Your Cloud Servers
Let's start from creating machines on our Playground. We will have 2 masters, 2 workers and 1 load balancer. 

![img](https://github.com/Bes0n/KubernetestheHardWay/blob/master/images/img2.png)

### Client Tools
In order to proceed with *Kubernetes the Hard Way*, there are some client tools that you need to install on your local workstation. These include cfssl and kubectl. This lesson introduces these tools and guides you through the process of installing them. After completing this lesson, you should have cfssl and kubectl installed correctly on your workstation.
  
You can find more information on how to install these tools, as well as instructions for OS X/Linux, here: https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-client-tools.md

**Commands used in the demo to install the client tools in a Linux environment:**
  
cfssl:
```
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
cfssl version
```
  
If you want to work on an i386 machine, use these commands to install cfssl instead:
```
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-386 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-386
chmod +x cfssl_linux-386 cfssljson_linux-386
sudo mv cfssl_linux-386 /usr/local/bin/cfssl
sudo mv cfssljson_linux-386 /usr/local/bin/cfssljson
cfssl version
```
  
kubectl:
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

## Provisioning the CA and Generating TLS Certificates
### Why Do We Need a CA and TLS Certificates?
In this section of the course, we will be creating the certificate authority and various TLS certificates that we will need later to set up the Kubernetes cluster. This lesson provides some background on why we need these certificates and what they will be used for in Kubernetes. After completing this lesson, you will have a basic understanding of the context for the tasks that will be performed in this section. These tasks will provide you with a better understanding of what you will be doing as you provision the certificate authority and generate the TLS certificates.

![img](https://github.com/Bes0n/KubernetestheHardWay/blob/master/images/img3.png)

### Provisioning the Certificate Authority
In order to generate the certificates needed by Kubernetes, you must first provision a certificate authority. This lesson will guide you through the process of provisioning a new certificate authority for your Kubernetes cluster. After completing this lesson, you should have a certificate authority, which consists of two files: `ca-key.pem` and `ca.pem`.
  
Here are the commands used in the demo:
```
cd ~/
mkdir kthw
cd kthw/
```
  
UPDATE: cfssljson and cfssl will need to be installed. To install, complete the following commands:
```
sudo curl -s -L -o /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
sudo curl -s -L -o /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
sudo curl -s -L -o /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
sudo chmod +x /bin/cfssl*
```
  
Use this command to generate the certificate authority. Include the opening and closing curly braces to run this entire block as a single command.
```
{

cat > ca-config.json << EOF
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

cat > ca-csr.json << EOF
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

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

### Generating Client Certificates
Now that you have provisioned a certificate authority for the Kubernetes cluster, you are ready to begin generating certificates. The first set of certificates you will need to generate consists of the client certificates used by various Kubernetes components. In this lesson, we will generate the following client certificates: `admin`, `kubelet` (one for each worker node), `kube-controller-manager`, `kube-proxy`, and `kube-scheduler`. After completing this lesson, you will have the client certificate files which you will need later to set up the cluster.
  
Here are the commands used in the demo. The command blocks surrounded by curly braces can be entered as a single command:
```
cd ~/kthw
```
  
Admin Client certificate:
```
{

cat > admin-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```
  
Kubelet Client certificates. Be sure to enter your actual cloud server values for all four of the variables at the top:
```
WORKER0_HOST=<Public hostname of your first worker node cloud server>
WORKER0_IP=<Private IP of your first worker node cloud server>
WORKER1_HOST=<Public hostname of your second worker node cloud server>
WORKER1_IP=<Private IP of your second worker node cloud server>

{
cat > ${WORKER0_HOST}-csr.json << EOF
{
  "CN": "system:node:${WORKER0_HOST}",
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${WORKER0_IP},${WORKER0_HOST} \
  -profile=kubernetes \
  ${WORKER0_HOST}-csr.json | cfssljson -bare ${WORKER0_HOST}

cat > ${WORKER1_HOST}-csr.json << EOF
{
  "CN": "system:node:${WORKER1_HOST}",
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${WORKER1_IP},${WORKER1_HOST} \
  -profile=kubernetes \
  ${WORKER1_HOST}-csr.json | cfssljson -bare ${WORKER1_HOST}

}
```
  
Controller Manager Client certificate:
```
{

cat > kube-controller-manager-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```
  
Kube Proxy Client certificate:
```
{

cat > kube-proxy-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```
  
Kube Scheduler Client Certificate:
```
{

cat > kube-scheduler-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```
  
As a result we will have ca.pem (Public key) and ca-key.pem (Private key) certificates

![img](https://github.com/Bes0n/KubernetestheHardWay/blob/master/images/img4.png)


### Generating the Kubernetes API Server Certificate
We have generated all of the the client certificates our Kubernetes cluster will need, but we also need a server certificate for the Kubernetes API. In this lesson, we will generate one, signed with all of the hostnames and IPs that may be used later in order to access the Kubernetes API. After completing this lesson, you will have a Kubernetes API server certificate in the form of two files called `kubernetes-key.pem` and `kubernetes.pem`.
  
Here are the commands used in the demo. Be sure to replace all the placeholder values in **CERT_HOSTNAME** with their real values from your cloud servers:

```
cd ~/kthw

CERT_HOSTNAME=10.32.0.1,<controller node 1 Private IP>,<controller node 1 hostname>,
<controller node 2 Private IP>,<controller node 2 hostname>,<API load balancer
Private IP>,<API load balancer hostname>,127.0.0.1,localhost,kubernetes.default

{

cat > kubernetes-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${CERT_HOSTNAME} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

### Generating the Service Account Key Pair
Kubernetes provides the ability for service accounts to authenticate using tokens. It uses a key-pair to provide signatures for those tokens. In this lesson, we will generate a certificate that will be used as that key-pair. After completing this lesson, you will have a certificate ready to be used as a service account key-pair in the form of two files: `service-account-key.pem` and `service-account.pem`.
  
Here are the commands used in the demo:
```
cd ~/kthw

{

cat > service-account-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

### Distributing the Certificate Files
Now that all of the necessary certificates have been generated, we need to move the files onto the appropriate servers. In this lesson, we will copy the necessary certificate files to each of our cloud servers. After completing this lesson, your controller and worker nodes should each have the certificate files which they need.
  
Here are the commands used in the demo. Be sure to replace the placeholders with the actual values from from your cloud servers.
  
Move certificate files to the worker nodes:
```
scp ca.pem <worker 1 hostname>-key.pem <worker 1 hostname>.pem user@<worker 1 public IP>:~/
scp ca.pem <worker 2 hostname>-key.pem <worker 2 hostname>.pem user@<worker 2 public IP>:~/
```
  
Move certificate files to the controller nodes:
```
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem user@<controller 1 public IP>:~/
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem user@<controller 2 public IP>:~/
```

### Creating a Certificate Authority and TLS Certificates for Kubernetes
**Provision the certificate authority (CA).**
- You can provision the certificate authority like so:
```
{

cat > ca-config.json << EOF
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

cat > ca-csr.json << EOF
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

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

**Generate the necessary Kubernetes client certs, as well as kubelet client certs for two worker nodes.**
- Use these commands to generate the client certs.
- Admin client cert:
```
{

cat > admin-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

- Kubelet client certs:
```
{
cat > worker0.mylabserver.com-csr.json << EOF
{
  "CN": "system:node:worker0.mylabserver.com",
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=172.34.1.0,worker0.mylabserver.com \
  -profile=kubernetes \
  worker0.mylabserver.com-csr.json | cfssljson -bare worker0.mylabserver.com

cat > worker1.mylabserver.com-csr.json << EOF
{
  "CN": "system:node:worker1.mylabserver.com",
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=172.34.1.1,worker1.mylabserver.com \
  -profile=kubernetes \
  worker1.mylabserver.com-csr.json | cfssljson -bare worker1.mylabserver.com

}
```

- Kube Controller Manager client cert:
```
{

cat > kube-controller-manager-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

- Kube Proxy client cert:
```


cat > kube-proxy-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

- Kube Scheduler client cert:
```
{

cat > kube-scheduler-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

**Generate the Kubernetes API server certificate.**
- You can generate the Kubernetes API server certificate like so:
```
{

cat > kubernetes-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,172.34.0.0,controller0.mylabserver.com,172.34.0.1,controller1.mylabserver.com,172.34.2.0,kubernetes.mylabserver.com,127.0.0.1,localhost,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

**Generate a Kubernetes service account key pair.**
- To generate the service account key pair, do the following:
```
{

cat > service-account-csr.json << EOF
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

## Generating Kubernetes Configuration Files for Authentication
### What are Kubeconfigs and Why Do We Need Them?
In this section, we will be generating kubeconfigs which will be used later to set up a Kubernetes cluster. Before proceeding, however, it is a good idea to have some basic understanding of what kubeconfigs are, and what purpose they serve. This lesson introduces you to kubeconfigs and provides some background on what they do and why they are needed. After completing this lesson, you will have an idea of how kubeconfigs are used, which will prepare you for the task of creating them.
  
You can find more information on kubeconfigs in the official Kubernetes documentation: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

### Generating Kubeconfigs for the Cluster
The next step in building a Kubernetes cluster the hard way is to generate kubeconfigs which will be used by the various services that will make up the cluster. In this lesson, we will generate these kubeconfigs. After completing this lesson, you should have a set of kubeconfigs which you will need later in order to configure the Kubernetes cluster.
  

![img](https://github.com/Bes0n/KubernetestheHardWay/blob/master/images/img5.png)

![img](https://github.com/Bes0n/KubernetestheHardWay/blob/master/images/img6.png)

- Here are the commands used in the demo. Be sure to replace the placeholders with actual values from your cloud servers.
  
- Create an environment variable to store the address of the Kubernetes API, and set it to the private IP of your load balancer cloud server:
```
KUBERNETES_ADDRESS=<load balancer private ip>
```

- Generate a kubelet kubeconfig for each worker node:
```
for instance in <worker 1 hostname> <worker 2 hostname>; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

- Generate a kube-proxy kubeconfig:
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

- Generate a kube-controller-manager kubeconfig:
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

- Generate a kube-scheduler kubeconfig:
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

- Generate an admin kubeconfig:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

### Distributing the Kubeconfig Files
Now that we have generated the kubeconfig files that we will need in order to configure our Kubernetes cluster, we need to make sure that each cloud server has a copy of the kubeconfig files that it will need. In this lesson, we will distribute the kubeconfig files to each of the worker and controller nodes so that they will be in place for future lessons. After completing this lesson, each of your worker and controller nodes should have a copy of the kubeconfig files it needs.

- Here are the commands used in the demo. Be sure to replace the placeholders with the actual values from your cloud servers.

- Move kubeconfig files to the worker nodes:
```
scp <worker 1 hostname>.kubeconfig kube-proxy.kubeconfig user@<worker 1 public IP>:~/
scp <worker 2 hostname>.kubeconfig kube-proxy.kubeconfig user@<worker 2 public IP>:~/
```

Move kubeconfig files to the controller nodes:
```
scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig user@<controller 1 public IP>:~/
scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig user@<controller 2 public IP>:~/
```