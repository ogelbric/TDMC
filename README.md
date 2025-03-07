# Tanzu Data Management Console (TDMC)


## Minimum requirements

```
TDMC Control Plane
1 Control Plane Best-Effort-Medium
4 Worker Nodes - Custom VM Class 8vCPUs 16GB RAM 30Gi for containerd 

TDMC Data Plane
1 Control Plane Best-Effort-Medium
5 Worker Nodes - Custom VM Class 8vCPUs 16GB RAM
```



## Steps for Shepeard env

```

1) Creating a new ENV
	sheepctl pool lock TKGS-3.0.0-VC-8u3-Non-Airgap -n Tanzu-Sales --lifetime 5d --description 'US East (Orf Gelbrich)'
	sheepctl pool lock TKGS-3.0.0-VC-8u3-Non-Airgap -n Tanzu-Sales --lifetime 5d --description 'US East (Orf Gelbrich)'  

2) Get Shepherd lock and IP/passwords

	sheepctl lock list -n Tanzu-Sales
	
	+--------------------------------------+--------+---------------+------------+-------------+------------------------------+------------------------------+
	|                  ID                  | STATUS |    CREATED    | EXPIRATION |  NAMESPACE  |             POOL             |         DESCRIPTION          |
	+--------------------------------------+--------+---------------+------------+-------------+------------------------------+------------------------------+
	| 423ad174-9b79-4e6e-9dcf-d3142eddfb66 | locked | 6 minutes ago | in 5 days  | Tanzu-Sales | TKGS-3.0.0-VC-8u3-Non-Airgap | US East (Orf Gelbrich)       |
	| adbd6c26-527c-43f4-9b24-c9b15dd4dde5 | locked | 8 days ago    | in 5 days  | Tanzu-Sales | TKGS-3.0.0-VC-8u3-Non-Airgap | US West/Partners (Bob Bauer) |
	| 2447c493-b8f3-44b7-811a-611c2b628998 | locked | 9 days ago    | in 5 days  | Tanzu-Sales | TKGS-3.0.0-VC-8u3-Non-Airgap | EMEA (Timo Salm)             |
	+--------------------------------------+--------+---------------+------------+-------------+------------------------------+------------------------------+

	#
	#lock id from above table
	#
	export LOCKID="423ad174-9b79-4e6e-9dcf-d3142eddfb66"
	sheepctl lock get -n Tanzu-Sales ${LOCKID} -j -o lockfile.json
	#
	# brew install jq on your machine!
	#
	export username=kubo
	export password=`sheepctl lock get ${LOCKID} -j |jq -r .outputs.vm.jumper.password`
	export jumpip=`sheepctl lock get ${LOCKID} -j |jq -r .outputs.vm.jumper.hostname`
	#


	export vcenterip=`sheepctl lock get ${LOCKID} -j |jq -r '.outputs.vm."vc.0".hostname'`
	#
	#if the IP starts with 192 then a proxy in the browser for vCenter access has to be used
	#
	if [[ ${vcenterip} == *"192"* ]] ; then echo "Use Proxy in Browser: ${jumpip}"; fi

	echo "————————————————————————————————————"
	echo "ssh ${username}@${jumpip}"
	echo "Password for everything: ${password}"
	echo "Jump IP / Proxy IP: ${jumpip}"
	echo "vCenterIP: ${vcenterip}"
	echo “———————————————————————————————-————"
	#
	# get API end point
	#
	sheepctl lock get ${LOCKID} -j |jq -r '.outputs.generic.cluster_config' | awk '{ print $8 }'  
       
	#’192.168.116.204'



3) ssh to jumper

	ssh ${username}@${jumpip}
	mkdir orf
	cd orf


4) Test connection to supervisor cluster 

	alias k=kubectl
	kubectl vsphere login --server=192.168.116.204 --vsphere-username administrator@vsphere.local --insecure-skip-tls-verify

	#
	# user password from above # RBGI+mwii_DT2fd7
	#

	kubectl config get-contexts
	CURRENT   NAME                   CLUSTER           AUTHINFO                                          NAMESPACE
	*         192.168.116.204        192.168.116.204   wcp:192.168.116.204:administrator@vsphere.local   
 	         svc-tkg-domain-c9      192.168.116.204   wcp:192.168.116.204:administrator@vsphere.local   svc-tkg-domain-c9
  	        svc-velero-domain-c9   192.168.116.204   wcp:192.168.116.204:administrator@vsphere.local   svc-velero-domain-c9
  	        testns                 192.168.116.204   wcp:192.168.116.204:administrator@vsphere.local   testns

	kubectl config use-context testns 

	kubectl get cluster
	
	kubectl delete cluster tkg-sm-non-airgapped
	#
	# errored out with some permission issue
	#
	#delte testns in vCenter to delete cluster 

	# create new namespace: namespace1000

	# create new vmclass: vm8cpu16ram 
	# make sure the new calls is in the namespace

	kubectl vsphere login --server=192.168.116.204 --vsphere-username administrator@vsphere.local --insecure-skip-tls-verify
	#RBGI+mwii_DT2fd7

	kubectl config use-context namespace1000

	kubo@JAuUH8JihQe9f:~/orf$ kubectl get virtualmachineclasses
	NAME                  CPU   MEMORY
	best-effort-2xlarge   8     64Gi
	best-effort-4xlarge   16    128Gi
	best-effort-8xlarge   32    128Gi
	best-effort-large     4     16Gi
	best-effort-medium    2     8Gi
	best-effort-small     2     4Gi
	best-effort-xlarge    4     32Gi
	best-effort-xsmall    2     2Gi
	guaranteed-2xlarge    8     64Gi
	guaranteed-4xlarge    16    128Gi
	guaranteed-8xlarge    32    128Gi
	guaranteed-large      4     16Gi
	guaranteed-medium     2     8Gi
	guaranteed-small      2     4Gi
	guaranteed-xlarge     4     32Gi
	guaranteed-xsmall     2     2Gi
	tpsm                  12    36Gi
	vm8cpu16ram           8     16Gi  <---

	#
	#And check tar as well
	#

	kubectl get tkr
	NAME                              VERSION                         READY   COMPATIBLE   CREATED
	v1.28.8---vmware.1-fips.1-tkg.2   v1.28.8+vmware.1-fips.1-tkg.2   True    True         4d2h
	v1.29.4---vmware.3-fips.1-tkg.1   v1.29.4+vmware.3-fips.1-tkg.1   True    True         4d2h

5) Create MGT cluster 

cat <<'EOF' > /home/kubo/orf/orfclustermgt.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mgtcluster1
  namespace: namespace1000
spec:
  clusterNetwork:
    services:
      # cidrBlocks: ["10.96.0.0/12"]
      cidrBlocks: ["192.168.0.0/20"]
    pods:
      cidrBlocks: ["192.168.16.0/20"]
      # cidrBlocks: ["192.168.16.0/20"]
    serviceDomain: "cluster.local"
  topology:
    class: tanzukubernetescluster
    version: v1.29.4---vmware.3-fips.1-tkg.1
    controlPlane:
      replicas: 1
    workers:
      machineDeployments:
        - class: node-pool
          name: node-pool-1
          replicas: 4
    variables:
      - name:  ntp
        value: "ntp.broadcom.net"
      - name: vmClass
        value: vm8cpu16ram
      - name: storageClass
        value: tkgs-k8s-obj-policy
      - name: defaultStorageClass
        value: tkgs-k8s-obj-policy
      - name: nodePoolVolumes
        value:
        - capacity:
            storage: "40Gi"
          mountPath: "/var/lib/containerd"
          name: containerd
          storageClass: tkgs-k8s-obj-policy
      - name: controlPlaneVolumes
        value:
        - capacity:
            storage: "4Gi"
          mountPath: "/var/lib/etcd"
          name: etcd
          storageClass: tkgs-k8s-obj-policy
EOF


	#
	# Create cluster (this will take some time ~30-40min / Watch in vCenter)
	#
	
	kubectl apply -f orfclustermgt.yaml


5) Create database cluster 

cat <<'EOF' > /home/kubo/orf/orfclusterdb1.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: dbcluster1
  namespace: namespace1000
spec:
  clusterNetwork:
    services:
      # cidrBlocks: ["10.96.0.0/12"]
      cidrBlocks: ["192.168.0.0/20"]
    pods:
      # cidrBlocks: ["192.168.16.0/20"]
      cidrBlocks: ["192.168.16.0/20"]
    serviceDomain: "cluster.local"
  topology:
    class: tanzukubernetescluster
    version: v1.29.4---vmware.3-fips.1-tkg.1  
    controlPlane:
      replicas: 1
    workers:
      machineDeployments:
        - class: node-pool
          name: node-pool-1
          replicas: 5
    variables:
      - name:  ntp
        value: "ntp.broadcom.net"
      - name: vmClass
        value: vm8cpu16ram
      - name: storageClass
        value: tkgs-k8s-obj-policy
      - name: defaultStorageClass
        value: tkgs-k8s-obj-policy
      - name: nodePoolVolumes
        value:
        - capacity:
            storage: "40Gi"
          mountPath: "/var/lib/containerd"
          name: containerd
          storageClass: tkgs-k8s-obj-policy
      - name: controlPlaneVolumes
        value:
        - capacity:
            storage: "4Gi"
          mountPath: "/var/lib/etcd"
          name: etcd
          storageClass: tkgs-k8s-obj-policy
EOF


	#
	# Create cluster (this will take some time ~30-40min / Watch in vCenter)
	#
	
	kubectl apply -f orfclusterdb1.yaml


6) Log onto MGT cluster
	alias k=kubectl
	kubectl vsphere login --server 192.168.116.204  --vsphere-username administrator@vsphere.local --tanzu-kubernetes-cluster-namespace namespace1000 --tanzu-kubernetes-cluster-name mgtcluster1 --insecure-skip-tls-verify

	#RBGI+mwii_DT2fd7

	kubectl config use-context mgtcluster1



7) Tanzu Data Management Console
 
	# Documentation
	# https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-data-management-console/1-0/tdmc/index.html
	#
	# Install doc
	# https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-data-management-console/1-0/tdmc/install.html
	#
	# Download
https://support.broadcom.com/group/ecx/productfiles?subFamily=Tanzu%20Data%20Management%20Console&displayGroup=Tanzu%20Data%20Management%20Console&release=1.0.0&os=&servicePk=527657&language=EN

	#
	# download these 2 files
	#

	#tdmc-cli-1.0.0-82f3a9.tar
	#tdmc-installer-1.0.0-82f3a9.tar

	#RBGI+mwii_DT2fd7

	#
	# scp tp jump box
	#
	scp tdmc-cli-1.0.0-82f3a9.tar ${username}@${jumpip}:/home/kubo/orf/.   
	scp tdmc-installer-1.0.0-82f3a9.tar ${username}@${jumpip}:/home/kubo/orf/.   

	tar xvf tdmc-installer-1.0.0-82f3a9.tar
	cd bin
	sudo install tdmc-installer-linux-amd64 /usr/local/bin/tdmc-installer

	# Init
	# one can select the GUI option here be I used the below yaml and CLI install 
	#
	tdmc-installer install

	#
	#
	# the config.yaml file for the install
	#
	#

cat <<'EOF' > /home/kubo/orf/config.yaml
EnvironmentDetails:
  Environment: tdmc
  StorageClass: tkgs-k8s-obj-policy
  Provider: tkg
  Insecure: false 
VSphereDetails:
  username: "administrator@vsphere.local"   
  password: "RBGI+mwii_DT2fd7"
  fqdn: "192.168.111.50"
  VSphereNamespace: "namespace1000"
  tkgClusterName: "mgtcluster1"
  supervisorIp: "192.168.116.204"
ControlPlaneSize:
  controlPlaneSize: tiny
CertificateDetails:
  generateSelfSignedCert: true
  configureCoreDns: true
    # if configureCoreDns: false, then cluster core dns won't be patched and it might lead to failure of communication between control plane and data plane pods
    # If generateSelfSignedCert: false, enter all the three certificateCA, certificateKey, certificateBody
    # Enter idpIp, operationQueueIp and controlPlaneIp in case you want to provide custom IP for the exposed services
  certificateCA:
  certificateKey:
  certificateBody:
  # Certificate Domain Should be wildcard certificate domain. Ex - *.tdmc.broadcom.com
  certificateDomain: "*.tanzu.platform.io"
  idpIp: ""
  operationQueueIp: ""
  controlPlaneIp: ""
#SmtpDetails:
  #SMTP Section is optional, we can remove this section if SMTP details are unavailable
  #host: smtp_host_goes_here
  #port: smtp_port_goes_here
  #from: smtp_from_goes_here
  #username: smtp_username_goes_here
  #password: smtp_password_goes_here
  #tlsEnabled: true_or_false
  #authEnabled: true_or_false
SreLoginDetails:
  username: "sre@tanzu.platform.io"
  password: "VMware1!"
ImageRegistryDetails:
  registryType: "JFrog"
  # If isAirGapped: false, provide registryUrl else we can omit that field
  # If Harbor, registryUrl must include project name, e.g. harbor.example.com/my-tdmc
  registryUrl: "tdmc.packages.broadcom.com/v1.0.0"
  registryCreds:  |-
    {
      "auths": {
        "https://tdmc.packages.broadcom.com/v1.0.0":{
        "auth":"b3JxxxxxxRDSXJEaU1CdFlYYkQzZHN6WTVtZ1pfdXFwQ0hXTHphZEpONHBMWEFEeElIU0xjR0dSVlhIUzBlcnNjZkFpLXB4S2dBczNBajBUZE9xRmJpVWZ2NWQtVjlHTmtiQWpCMHF2WWIwMnVqSExJX0hOM2o1OWNodXQ5ZzR3",
        "username": "orf.gelbrich@broadcom.com",
        "password":"eyJ2ZXIiOxxxxxxXbD3dszY5mgZ_uqpCHWLzadJN4pLXADxIHSLcGGRVXHS0erscfAi-pxKgAs3Aj0TdOqFbiUfv5d-V9GNkbAjB0qvYb02ujHLI_HN3j59chut9g4w"
        }
      }
    }
EOF


#
#From the Broadcom down load section the green token ikon
#

Configure:
Repository Key: tdmc-docker-remote (or preferred on-premise naming convention)
URL: https://tdmc.packages.broadcom.com
User Name: orf.gelbrich@broadcom.com
Password / Access Token: eyJ2ZXIiOiIyIiwidHlxxxxxxztIVF3dCIrDiMBtYXbD3dszY5mgZ_uqpCHWLzadJN4pLXADxIHSLcGGRVXHS0erscfAi-pxKgAs3Aj0TdOqFbiUfv5d-V9GNkbAjB0qvYb02ujHLI_HN3j59chut9g4w
Click Test to confirm that the credentials are working.
Click Save & Finish.


This generates....

# original # ./credential-generator --url "tdmc.packages.broadcom.com/v1.0.0" --username "<USERNAME>" --password "<PASSWORD>"

./bin/credential-generator-linux-386 --url "tdmc.packages.broadcom.com/v1.0.0" --username "orf.gelbrich@broadcom.com" --password "eyJ2xxxxxxjHLI_HN3j59chut9g4w"


This....

{
	"auths": {
	  "https://tdmc.packages.broadcom.com/v1.0.0": {
		"auth":"b3JmLmdlbGJyaWNoQxxxxxx0hXTHphZEpONHBMWEFEeElIU0xjR0dSVlhIUzBlcnNjZkFpLXB4S2dBczNBajBUZE9xRmJpVWZ2NWQtVjlHTmtiQWpCMHF2WWIwMnVqSExJX0hOM2o1OWNodXQ5ZzR3",
		"username": "orf.gelbrich@broadcom.com",
		"password":"eyJ2ZXIiOiIyIixxxxxADxIHSLcGGRVXHS0erscfAi-pxKgAs3Aj0TdOqFbiUfv5d-V9GNkbAjB0qvYb02ujHLI_HN3j59chut9g4w"
	  }
	}
}

Reformat (take out tabs and extra spaces and place into EOF...EOF section)

#
# this takes some time
#

tdmc-installer install -f /home/kubo/orf/config.yaml



#
# to get status (this will be displayed at the start of the install keep track of it!
#
tdmc-installer status --id gajppY
tdmc-installer status --id gajppY --details true
tdmc-installer status --id YSHJUA --details true
tdmc-installer status --id YSHJUA --details true | grep Status
tdmc-installer status --id ZfWQd1 --details true | grep Status


#
# delte the installation (incase of config file issues (in my case 2 spaces issue and double quote issues)
#

tdmc-installer delete -f /home/kubo/orf/config.yaml

#
# outcome from install 
#
[*] Successfully Registered SRE User To Control Plane
  - Configure Core DNS Server
Received domain with `*.`.  HostedZone: tanzu.platform.io 
Patching CoreDNS kube-system/coredns
Created ConfigMap tdh-dns in namespace kube-system
Patching CoreDNS deployment coredns in namespace kube-system
Successfully Patched CoreDNS deployment coredns in namespace kube-system
Successfully patched CoreDNS kube-system/coredns
All Charts Are Deployed Successfully and Control Plane is functional. 
 In order to Access the control plane, please publish the following dns records with their associated IPs to access the control plane publicly
[IDP: 192.168.116.208  idp-tdmc.tanzu.platform.io]
[Control Plane: 192.168.116.208  tdmc-cp-tdmc.tanzu.platform.io]
[Operations Queue: 192.168.116.208  ops-queue-tdmc.tanzu.platform.io]
Register below DNS Nameserver details for accessing from outside of cluster
[DNSServer TCP: 192.168.116.207]
[DNSServer UDP: 192.168.116.209]
 [*] All Charts Deployed Successfully Service Ready to be Used
End Time of Install: 2025-03-06 18:26:26.881889677 +0000 UTC m=+2310.924328250



#
# extend the lock
#
sheepctl lock extend 423ad174-9b79-4e6e-9dcf-d3142eddfb66 -t 5d -n Tanzu-Sales 



#
# Change the IP in DNS to the k get svc -A | grep Load | grep -v "53:" | awk '{ print $5 }'  address 
#
 vi /etc/dnsmasq.d/vlan-dhcp-dns.conf
address=/tanzu.platform.io/192.168.116.208
#
# restart DNS
#
sudo systemctl restart dnsmasq



nslookup tdmc-cp-tdmc.tanzu.platform.io

#
# go to proxy enabled (jump box IP) browser
#
https://tdmc-cp-tdmc.tanzu.platform.io
#
# log in with config.yaml credential
#

```
## TDMC GUI

![Version](https://github.com/ogelbric/TDMC/blob/main/TDMC1.png)


## Broadcom Support Token

![Version](https://github.com/ogelbric/TDMC/blob/main/Token1.png)

