---
title: "Openshift IPSEC N/S"
date: 2025-_6-17
tags:
- openshift
categories:
- openshift
---

{{< admonition >}}
  In this POC, we will cover how to configure an IPSEC communication from OCP to a RHEL node
{{< /admonition >}}

### Configuration RHEL node side

#### Requirements

butane
libreswan

```bash
dns install -y butane libreswan
```

#### Create CA and certs

{{< admonition >}}
  For simplicty, we'ell use the same certificate with SAN matching both side
{{< /admonition >}}

```
mkdir ca certs private
openssl genrsa -out ca/ca.key.pem 2048
openssl req -x509 -new -nodes -key ca/ca.key.pem -sha256 -days 3650 -out ca/ca.crt.pem -subj "/CN=IPsec Test CA/O=MyOrg/OU=MyUnit/L=MyCity/ST=MyState/C=US"
openssl genrsa -out private/hosts.key.pem 2048

cat > openssl.cnf <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[ req_distinguished_name ]
C = US
ST = MyState
L = MyCity
O = MyOrg
OU = MyUnit
CN = worker-n.example.com

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = worker-n.example.com #CHANGEME
DNS.2 = rhel-remote.example.com #CHANGEME
EOF

openssl req -new -key private/hosts.key.pem -out certs/hosts.csr.pem -config openssl.cnf

openssl x509 -req -in certs/hosts.csr.pem -CA ca/ca.crt.pem -CAkey ca/ca.key.pem -CAcreateserial -out certs/hosts.crt.pem -days 365 -sha256 -extfile <(printf "subjectAltName=DNS:openshift-host.example.com,DNS:rhel-host.example.com")


openssl pkcs12 -export -out certs/hosts.p12 -inkey private/hosts.key.pem -in certs/hosts.crt.pem -certfile ca/ca.crt.pem -name "allnodes"

```



#### Import cert on RHEL node

```
ipsec initnss
ipsec import certs/hosts.p12

# Verify import if successfull with the name provided ; here "allnodes"
certutil -L -d sql:/var/lib/ipsec/nss/
Certificate Nickname                                         Trust Attributes
                                                             SSL,S/MIME,JAR/XPI

allnodes                                                     u,u,u
IPsec Test CA - MyOrg                                        CT,


```

#### IPSEC conf 

```
cat > /etc/ipsec.d/poc.conf << EOF
conn my-host-to-host-vpn
    left=10.8.109.144 #  CAHNGEME This is the RHEL node IP
    leftid=%fromcert
    leftcert="allnodes"    
    right=10.8.51.222 # CHANGEM This the OCP node IP
    rightid=%fromcert
    authby=rsasig              
    ikev2=yes
    ike=aes256-sha2      
    ikelifetime=8h                     
    esp=aes_gcm256
    auto=start                  
    type=transport 
EOF
```

#### Start ipsec.service

```
systemctl start ipsec.service
```


### Configration OCP side



#### Requirements

{{< admonition >}}
  Install nmstate operator
{{< /admonition >}}

#### NNCP 

```
cat > ipsec_nncp.yaml << EOF
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ipsec-config
spec:
  nodeSelector:
    kubernetes.io/hostname: mmayeras-tqhkk-worker-0-b4ktk
  desiredState:
    interfaces:
    - name: test-ipsec
      type: ipsec
      libreswan:
        left: worker-n.example.com # CHANGEME
        leftid: '%fromcert'
        leftrsasigkey: '%cert'
        leftcert: left_server
        leftmodecfgclient: false
        right: rhel-host.example.com # CHANGEME
        rightid: '%fromcert'
        rightrsasigkey: '%cert'
        rightsubnet: 10.x.x.x/xx # CHANGEME
        ikev2: insist
        type: transport
        esp: aes_gcm256
        ike: aes256-sha2;dh20
EOF

oc apply -f ipsec_nncp.yaml 
```


#### Machine config for certificate importation

```
cat > 99-ipsec-worker-endpoint-config.bu << EOF
  variant: openshift
  version: 4.18.0
  metadata:
    name: 99-worker-import-certs
    labels:
      machineconfiguration.openshift.io/role: worker
  systemd:
    units:
    - name: ipsec-import.service
      enabled: true
      contents: |
        [Unit]
        Description=Import external certs into ipsec NSS
        Before=ipsec.service

        [Service]
        Type=oneshot
        ExecStart=/usr/local/bin/ipsec-addcert.sh
        RemainAfterExit=false
        StandardOutput=journal

        [Install]
        WantedBy=multi-user.target
  storage:
    files:
    - path: /etc/pki/certs/ca.pem
      mode: 0400
      overwrite: true
      contents:
        local: ca/ca.crt.pem
    - path: /etc/pki/certs/hosts.p12
      mode: 0400
      overwrite: true
      contents:
        local: certs/hosts.p12
    - path: /usr/local/bin/ipsec-addcert.sh
      mode: 0740
      overwrite: true
      contents:
        inline: |
          #!/bin/bash -e
          echo "importing cert to NSS"
          certutil -A -n "CA" -t "CT,C,C" -d /var/lib/ipsec/nss/ -i /etc/pki/certs/ca.pem
          pk12util -W "" -i /etc/pki/certs/hosts.p12 -d /var/lib/ipsec/nss/
          certutil -M -n "allnodes" -t "u,u,u" -d /var/lib/ipsec/nss/
EOF

butane -d . 99-ipsec-worker-endpoint-config.bu -o ./99-ipsec-worker-endpoint-config.yaml

oc apply -f 
``` 

{{< admonition warning >}}
  Ensure thne name used in the certutil command match the name used in the openssl pkcs12 -export command used previsouly 
{{< /admonition >}}

{{< admonition >}}
  Waint until end of machine config rollout
{{< /admonition >}}


### Check config

#### Ensure tunnel is active

After rollout OCP side, the tunnel should be up

```
[root@bastion ipsec]# ipsec status
000 Total IPsec connections: loaded 1, active 1
000
000 State Information: DDoS cookies not required, Accepting new IKE connections
000 IKE SAs: total(1), half-open(0), open(0), authenticated(1), anonymous(0)
000 IPsec SAs: total(2), authenticated(2), anonymous(0)
000
000 #1: "my-host-to-host-vpn":500 STATE_V2_ESTABLISHED_IKE_SA (established IKE SA); REKEY in 27456s; REPLACE in 28087s; newest; idle;
000 #2: "my-host-to-host-vpn":500 STATE_V2_ESTABLISHED_CHILD_SA (established Child SA); REKEY in 27499s; REPLACE in 28087s; IKE SA #1; idle;
000 #2: "my-host-to-host-vpn" esp.248d1071@10.8.51.222 esp.4e68b7a9@10.8.109.144 Traffic: ESPin=0B ESPout=0B ESPmax=2^63B
000 #3: "my-host-to-host-vpn":500 STATE_V2_ESTABLISHED_CHILD_SA (established Child SA); REKEY in 27080s; REPLACE in 28100s; newest; eroute owner; IKE SA #1; idle;
000 #3: "my-host-to-host-vpn" esp.a3a6c6eb@10.8.51.222 esp.8fd9bdcd@10.8.109.144 Traffic: ESPin=128B ESPout=128B ESPmax=2^63B
000
000 Bare Shunt list:
000

[root@bastion ipsec]# ipsec trafficstatus
006 #2: "my-host-to-host-vpn", type=ESP, add_time=1750173085, inBytes=0, outBytes=0, maxBytes=2^63B, id='C=US, ST=MyState, L=MyCity, O=MyOrg, OU=MyUnit, CN=worker-n'
006 #3: "my-host-to-host-vpn", type=ESP, add_time=1750173098, inBytes=128, outBytes=128, maxBytes=2^63B, id='C=US, ST=MyState, L=MyCity, O=MyOrg, OU=MyUnit, CN=worker-n'
```

If not, try to enable it again until 
000 Total IPsec connections: loaded 1, active 1


```
ipsec auto --up my-host-to-host-vpn
ipsec auto --status
```


#### Verify with a packet capture


Run a tcpdump capture on one of the nodes (OCP node or RHEL node) 

{{< admonition >}}
  The Encapsulation Security Payload (ESP) is defined in RFC 4303, has IP protocol number 50
{{< /admonition >}}

Then ping from node to the other

```
tcpdump -i any -n -s 0 'ip proto 50'

tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes

17:45:53.567216 ens192 In  IP 10.8.51.222 > 10.8.109.144: ESP(spi=0x4c5a8df6,seq=0xc), length 100
17:45:53.567316 ens192 Out IP 10.8.109.144 > 10.8.51.222: ESP(spi=0xca19c2c5,seq=0xc), length 100
17:45:54.568259 ens192 In  IP 10.8.51.222 > 10.8.109.144: ESP(spi=0x4c5a8df6,seq=0xd), length 100
17:45:54.568334 ens192 Out IP 10.8.109.144 > 10.8.51.222: ESP(spi=0xca19c2c5,seq=0xd), length 100
^C
``` 



