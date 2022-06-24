# Two node ETCD cluster setup using virtualbox on windows 10.

## Follow the video and install etcd and etcdctl binaries on both VMs.

## Follow the video to download the cfssl and cfssljson binaries, then follow the commads below.

## On first node

```ruby
chmod 777 cfssl_1.6.1_linux_amd64 cfssljson_1.6.1_linux_amd64
sudo mv cfssl_1.6.1_linux_amd64 /usr/local/bin/cfssl
sudo mv cfssljson_1.6.1_linux_amd64 /usr/local/bin/cfssljson
```

# On first node
nano ca-config.json

{
    "signing": {
        "default": {
            "expiry": "8760h"
        },
        "profiles": {
            "etcd": {
                "expiry": "8760h",
                "usages": ["signing","key encipherment","server auth","client auth"]
            }
        }
    }
}

## On first node
nano ca-csr.json

{
  "CN": "etcd cluster",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "DE",
      "L": "Germany",
      "O": "Kubernetes",
      "OU": "ETCD-CA",
      "ST": "Berlin"
    }
  ]
}

## On first node
cfssl gencert -initca ca-csr.json | cfssljson -bare ca


## On both nodes
export ETCD1_IP="<your IP of the first server>"
export ETCD2_IP="<your IP of the second server>"

## On first node
nano etcd-csr.json
  
{
  "CN": "etcd",
  "hosts": [
    "localhost",
    "127.0.0.1",
    "${ETCD1_IP}",
    "${ETCD2_IP}"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "DE",
      "L": "Germany",
      "O": "Kubernetes",
      "OU": "etcd",
      "ST": "Berlin"
    }
  ]
}

## On first node
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd etcd-csr.json | cfssljson -bare etcd



## On both nodes
export NODE_IP="<IP address of the node where this command runs>"
export ETCD_NAME=$(hostname -s)


## On both nodes
[Unit]
Description=etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd
  --name ${ETCD_NAME}
  --cert-file=/etc/etcd/ssl/etcd.pem
  --key-file=/etc/etcd/ssl/etcd-key.pem
  --peer-cert-file=/etc/etcd/ssl/etcd.pem
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem
  --trusted-ca-file=/etc/etcd/ssl/ca.pem
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem
  --peer-client-cert-auth
  --client-cert-auth
  --initial-advertise-peer-urls https://${NODE_IP}:2380
  --listen-peer-urls https://${NODE_IP}:2380
  --advertise-client-urls https://${NODE_IP}:2379
  --listen-client-urls https://${NODE_IP}:2379,https://127.0.0.1:2379
  --initial-cluster-token etcd-cluster-1
  --initial-cluster etcd1=https://${ETCD1_IP}:2380,etcd2=https://${ETCD2_IP}:2380
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target


  
## On both nodes
systemctl daemon-reload
systemctl enable --now etcd


## On both nodes
export ETCDCTL_API=3 
export ETCDCTL_ENDPOINTS=https://192.168.178.116:2379,https://192.168.178.115:2379
export ETCDCTL_CACERT=/etc/etcd/ssl/ca.pem
export ETCDCTL_CERT=/etc/etcd/ssl/etcd.pem
export ETCDCTL_KEY=/etc/etcd/ssl/etcd-key.pem


## On both nodes
etcdctl member list
etcdctl endpoint status
etcdctl endpoint health
