# Installing baremetal sno (single node openshift) ocp disconnected step-by-step
Setup server
```
#register system
subscription-manager register
subscription-manager auto-attach

#update system
sudo dnf update
sudo dnf install /usr/bin/nmstatectl -y

#grab oc 
#rhel9
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux-amd64-rhel9.tar.gz
#rhel8
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux-amd64-rhel8.tar.gz

#grab openshift-install
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz

#Extract the client and place in into the path
sudo tar xzf openshift-client-linux*.tar.gz -C /usr/local/sbin/ oc kubectl

sudo tar xzf openshift-install-linux.tar.gz -C /usr/local/sbin openshift-install

#validate client and install version and platform
oc version
openshift-install version
```
:::info
Ensure architecture matches the target destination aarch64 for arm or amd64 for x86_64
:::
```
#set hostname to  external dns name 
sudo  nmcli general hostname openshift-repo.example.com

```

Making an offline installation bundle for OpenShift requires [mirroring/downloading the container images](https://docs.openshift.com/container-platform/4.15/installing/disconnected_install/installing-mirroring-disconnected.html) and then [hosting those container images in a container registry](https://docs.openshift.com/container-platform/4.15/installing/disconnected_install/installing-mirroring-creating-registry.html) that is accessible by the cluster nodes. The download process can put the container images into the local filesystem or upload them directly into the container registry. (a USB stick, a directory that will be burnt to a DVD, or a folder that will be uploaded into S3 or similar storage)

:::info
A minimal download of OpenShift 4.15 requires ~15GB of space.

A minimal download of OpenShift Platform Plus requires ~50GB of space.

A full download of All OpenShift Operators and versions can be close to 1TB.
:::

Lots of good information in this blog - [Mirroring OpenShift Registries: The Easy Way by Ben Schmaus and Daniel Messer (August 23, 2022)](https://cloud.redhat.com/blog/mirroring-openshift-registries-the-easy-way).

## Create DNS Entries
Create necessary dns entries for use:

mirror-registry.example.com

master-0.example.com

api.snow-cluster.example.com

*.apps.snow-cluster.example.com

## Install `mirror-registry` (aka mini Quay)
:::info
Recommend running these commands as root then you can login as any user
:::

```
#pull down and extra mirror-registry
wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz

tar xvzf mirror-registry.tar.gz

sudo ./mirror-registry install --help


#setup
### password must be at least 8 characters and contain no whitespace
sudo ./mirror-registry install --quayRoot /data/mirror-registry --quayStorage /data/mirror-registry --initUser admin --initPassword 3yHyHFb9ELEavGixZG846
```
:::info
INFO[2023-04-20 14:39:12] Quay installed successfully, config data is stored in ~/quay-install
INFO[2023-04-20 14:39:12] Quay is available at https://openshift-repo.example.com:8443 with credentials (admin, 3yHyHFb9ELEavGixZG846)
:::
```

#copy certificates to be trusted
sudo cp -v /data/mirror-registry/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

#****optional cleanup if restarting****
sudo rm /etc/pki/ca-trust/source/anchors/rootCA.pem
sudo update-ca-trust
#****optional cleanup if restarting****

#test login
podman login -u admin -p 3yHyHFb9ELEavGixZG846 $(hostname -f):8443
```
### Starting and stopping the `mirror-registry`

Running the `./mirror-registry install ...` results in several `systemd` services being created. You can see which services were created like this:

```
systemctl -a | grep quay
  quay-app.service         loaded    active   running   Quay Container
  quay-pod.service         loaded    active   exited    Infra Container for Quay
  quay-postgres.service    loaded    active   running   PostgreSQL Podman Container for Quay
  quay-redis.service       loaded    active   running   Redis Podman Container for Quay
```

These services will automatically start when the system is rebooted. The `quay-redis`, `quay-db`, and `quay-app` services depend on the `quay-pod` service. You can restart everything with one command:

```
systemctl restart quay-pod
```
## Install the `oc-mirror` plugin

```
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/oc-mirror.tar.gz

mkdir -p $HOME/.local/bin
tar xvzf ./oc-mirror.tar.gz -C $HOME/.local/bin
chmod a+x $HOME/.local/bin/oc-mirror

oc plugin list
oc mirror --help
```

## Add mirror-registry credentials to pull-secret

Download your "pull secret" from the Red Hat OpenShift Cluster Manager at https://console.redhat.com/openshift/install/pull-secret.

```
jq . pull-secret.txt
mkdir -p $HOME/.docker
mv -v pull-secret.txt $HOME/.docker/config.json
podman login -u admin -p 3yHyHFb9ELEavGixZG846 --authfile $HOME/.docker/config.json ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443

# OPTIONAL - remove the Insights & Telemetry connection
# https://docs.openshift.com/container-platform/4.12/support/remote_health_monitoring/opting-out-of-remote-health-reporting.html
 podman logout --authfile $HOME/.docker/config.json cloud.openshift.com

# Confirm contents and create a backup
jq . $HOME/.docker/config.json
cp -v $HOME/.docker/config.json ~/pull-secret.json
```

## Setup mirror registry file
```
oc-mirror list releases
oc-mirror list releases --version 4.15 --channels

oc-mirror list operators
oc-mirror list operators --version 4.15 --catalogs
oc-mirror list operators --version 4.15 \
    --catalog registry.redhat.io/redhat/redhat-operator-index:v4.15 \
    --package odf-operator \
    --channel stable-4.15


oc-mirror init | tee imageset-config.yaml
vi imageset-config.yaml

#find channels to slim down
podman pull registry.redhat.io/redhat/redhat-operator-index:v4.15
podman unshare
cd $(podman image mount registry.redhat.io/redhat/redhat-operator-index:v4.15)
ls configs

jq .name configs/*/catalog.json

# browse the metadata of an Operator
jq . configs/advanced-cluster-management/catalog.json | less -i

#list channels
jq '.schema, .name' configs/advanced-cluster-management/catalog.json

#When done exit the POD
exit
```
## Sample [imageset-config.yaml](https://raw.githubusercontent.com/pekramp/notes/main/imageset-config.yaml?token=GHSAT0AAAAAACOYPAFY4UBYFHCRKZNL7GDYZO44ITQ)
```
---
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
archiveSize: 4
mirror:
  platform:
    architectures:
      - amd64
    channels:
    - name: stable-4.15
      type: ocp
      minVersion: 4.15.12
    graph: true
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.15
    packages:
    - name: quay-bridge-operator
      channels:
        - name: stable-3.10
    - name: quay-operator
      channels:
        - name: stable-3.10
    - name: cincinnati-operator
      channels:
        - name: v1
    - name: cluster-logging
      channels:
        - name: stable-5.8
    - name: compliance-operator
      channels:
        - name: stable
    - name: web-terminal
      channels:
        - name: fast
    - name: file-integrity-operator
      channels:
        - name: stable
    - name: ocs-operator
      channels:
        - name: stable-4.15
    - name: local-storage-operator
      channels:
        - name: stable
  additionalImages:
  - name: registry.redhat.io/rhel9/rhel-guest-image:latest
  - name: registry.redhat.io/rhel8/rhel-guest-image:latest
```

## Run the oc-mirror
If you are partially disconnected do the following:
```
#prerun 
#df -h
#/ avail 88G

#fill the registry
time oc mirror --config=imageset-config.yaml docker://openshift-repo.example.com:8443/415-mirror

#time 
#info: Mirroring completed in 19m53.06s (56.23MB/s)
#Rendering catalog image "openshift-repo.example.com:8443/415-mirror/redhat/redhat-operator-index:v4.12" with file-based catalog
#Writing image mapping to oc-mirror-workspace/results-1682046532/mapping.txt
#Writing UpdateService manifests to oc-mirror-workspace/results-1682046532
#Writing CatalogSource manifests to oc-mirror-workspace/results-1682046532
#Writing ICSP manifests to oc-mirror-workspace/results-1682046532

#real    25m39.673s
#user    6m9.616s
#sys     2m59.360s


#postrun
#/ avail 25G

```
If you are fully disconnected do the following:
```
oc mirror --config=imageset-config.yaml file://<path_to_output_directory> 

ls <path_to_output_directory>
mirror_seq1_000000.tar
```
Copy to disconnected environment through approved channels, then on disconnected side:
```
oc mirror --from=./mirror_seq1_000000.tar docker://openshift-repo.example.com:8443/415-mirror 
```

## Get the mirror info for install
```
#cat mirror file
cat oc-mirror-workspace/results-1682046532/imageContentSourcePolicy.yaml
---
- mirrors:
  - openshift-repo.example.com:8443/415-mirror/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - openshift-repo.example.com:8443/415-mirror/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release


```
## Create install-config
```
openshift-install create install-config --dir <installation_directory> 
```

## Customize install-config
Customize resources
Add mirror information 
Add additionalTrustedBundle
Confirm PullSecret has mirror registry info

## Sample [install-config](https://raw.githubusercontent.com/pekramp/notes/main/install-config.yaml?token=GHSAT0AAAAAACOYPAFZNLWR7HOAUKKPLDXGZO4424Q)
```
---
additionalTrustBundlePolicy: Always
apiVersion: v1
baseDomain: example.com
compute:
- architecture: amd64
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  name: master
  platform:
  replicas: 1
metadata:
  creationTimestamp: null
  name: sno-cluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3B.............."pkramp@redhat.com"}}}'
sshKey: |
  ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMnrXX3IFuW0mCL1VGiDhjrvOG9AeE0jydJaeuodugX2Enbl/mC8tpBUrUrsGT68jPB1FOe3JgRJHqIbB4jYKwc= ec2-user@ec2-18-216-214-86.us-east-2.compute.amazonaws.com
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIEPTCCAyWgAwIBAgIUX7jN/sfBvpDPPi+hwpc10UcB2h8wDQYJKoZIhvcNAQEL
  BQAwgYsxCzAJBgNVBAYTAlVTMQswCQYDVQQIDAJWQTERMA8GA1UEBwwITmV3IFlv
  cmsxDTALBgNVBAoMBFF1YXkxETAPBgNVBAsMCERpdmlzaW9uMTowOAYDVQQDDDFl
  YzItMTgtMjE2LTIxNC04Ni51cy1lYXN0LTIuY29tcHV0ZS5hbWF6b25hd3MuY29t
  MB4XDTIzMDQyMDIyNDk0NloXDTI2MDIwNzIyNDk0NlowgYsxCzAJBgNVBAYTAlVT
  MQswCQYDVQQIDAJWQTERMA8GA1UEBwwITmV3IFlvcmsxDTALBgNVBAoMBFF1YXkx
  ETAPBgNVBAsMCERpdmlzaW9uMTowOAYDVQQDDDFlYzItMTgtMjE2LTIxNC04Ni51
  cy1lYXN0LTIuY29tcHV0ZS5hbWF6b25hd3MuY29tMIIBIjANBgkqhkiG9w0BAQEF
  AAOCAQ8AMIIBCgKCAQEA60xdQlOcVH8Hh/de5hZjypLa3OMEEoRXOM7jN9XRaUXr
  VhywW48b5RSSufPUBPDIrN+WxNlHG1nydOQF0jyxOte8r3qDes71WZNRaOOZzyDT
  rwC2CUonw9zxgnbgLKt148E1DFVP0DPvPJA3lwbbIUGRR16BOG+SbNFRwsdRBYJ7
  AoYBag+akruUfHSAoqk5prOOsbb0LMZT71baiIZctIBDvo3xlvIF1ZWhc5Xm1D4o
  M4HO0nczA/MNpbFRpu/xTlnm28103ADBEOLxI6/Bx5NOBPA4YwMBsX2VHd+hfc0/
  PcsyZiC9y4MMAEvxqfFllIDncLQPns4v52SoK0ERFQIDAQABo4GWMIGTMAsGA1Ud
  DwQEAwIC5DATBgNVHSUEDDAKBggrBgEFBQcDATA8BgNVHREENTAzgjFlYzItMTgt
  MjE2LTIxNC04Ni51cy1lYXN0LTIuY29tcHV0ZS5hbWF6b25hd3MuY29tMBIGA1Ud
  EwEB/wQIMAYBAf8CAQEwHQYDVR0OBBYEFP0++4S6z3Ytn3hmBDaoEYqcuxlIMA0G
  CSqGSIb3DQEBCwUAA4IBAQA0514SEnp88b+hq2ADANmPq84cEFq2nO86T7FpKEhq
  bqH+T8zhGk015Nl4Yca3g7E9RnRROrrRA9bWInNy6C+CwVsxyTIHoc2uOzGjS+cr
  9Tb1D4eCZ8ok4KHrZks3GkGYq8c62vFfee10Aj3x25fSiAzakD1q8H4IXOcvRDF+
  s8A4EHhwu0mVXASFZ32SIEV+dKYbTWV4r8NCFd/aEJI6w1GilBfKhhnBl57hS8ZW
  wvcXj6HTtLmSx65ScykWB1Fp0Gbzuvt9DtpPJ/OFe48AGECmKwp5HEB+lLouf6kJ
  NRsUZrye9cnovP8yIDVReSL1Bg+3/kvIpmNeOv6Lu9Hs
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - openshift-repo.example.com:8443/415-mirror/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - openshift-repo.example.com:8443/415-mirror/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release

```
## Create agent-config
```
openshift-install agent create agent-config-template
```
## Sample [agent-config](https://raw.githubusercontent.com/pekramp/notes/main/agent-config.yaml?token=GHSAT0AAAAAACOYPAFYXUJ4CRZDHTYHRMH2ZO45NJQ)
```
---
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: sno-cluster
rendezvousIP: 192.168.111.80 
hosts: 
  - hostname: master-0 
    interfaces:
      - name: eno1
        macAddress: 00:ef:44:21:e6:a5
    rootDeviceHints: 
      deviceName: /dev/sdb
    networkConfig: 
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:ef:44:21:e6:a5
          ipv4:
            enabled: true
            address:
              - ip: 192.168.111.80
                prefix-length: 23
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.111.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.111.2
            next-hop-interface: eno1
            table-id: 254
```
## Backup install-config && agent-config
:::info
****Copy to a backup directory before running, files are consumed****
:::

## Make the agent-based ISO image and boot
```
mkdir ocp415install/
cp ocp415install-backup/install-config.yaml ocp415install/install-config.yaml

cp ocp415install-backup/agent-config.yaml ocp415install/agent-config.yaml

#create iso
openshift-install --dir=ocp415install agent create image
#use iso to boot target box

#watch install boostrapping
openshift-install --dir=ocp415install agent wait-for bootstrap-complete --log-level=debug
#watch install complete
openshift-install --dir=ocp415install agent wait-for install-complete --log-level=debug

#OUTPUT
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/myuser/ocp415install/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.sno-cluster.example.com
INFO Login to the console with user: "kubeadmin", and password: "qrPRj-nWGSZ-zjeAU-qWQw9"

#Can export KUBECONFIG during install and monitor nodes and cluster operators coming up
oc get nodes
oc get co
```


### Disable the default OperatorHub Catalog Sources

```
oc patch OperatorHub cluster --type merge \
  --patch '{"spec":{"disableAllDefaultSources":true}}'
```
### Create a new disconnected Operator Catalog

```
oc get catalogsource --all-namespaces
#No resources found

oc create -f oc-mirror-workspace/results-1682046532/catalogSource-redhat-operator-index.yaml

#catalogsource.operators.coreos.com/redhat-operator-index created
```

### Troubleshooting
```
oc create configmap trusted-ca-list --from-file=ca-bundle.crt=rootCA.pem -n openshift-config

oc patch proxy/cluster --type=merge -p '{"spec":{"trustedCA":{"name":"trusted-ca-list"}}}'

#Wait for reboot

#Check pullsecret 
oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' >mypull.json

#Copy
cp mypull.json my-updated-pull.json

#If missing can add with 
oc registry login --registry="https://openshift-repo.example.com:8443" --auth-basic="admin:3yHyHFb9ELEavGixZG846" --to=my-updated-pull.json

#Replace
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=my-updated-pull.json

#delete catalog source
oc get catalogsource --all-namespaces
oc delete catalogsource redhat-operator-index -n openshift-marketplace

```

## Cleanup and restart

I wanted to find a way to delete all of the Quay images and start over, without changing my CA certificate, or doing `./mirror-registry uninstall ...` I found that I could create a Quay "super user" token and use that on the command line. [Apparently tokens are deprecated](https://docs.quay.io/glossary/access-token.html), but I couldn't figure out how to make a Robot Account work for me. First create a new Organization. I called mine "adminorg", then [follow the instructions to create a token](https://access.redhat.com/documentation/en-us/red_hat_quay/3/html/red_hat_quay_api_guide/using_the_red_hat_quay_api#create_oauth_access_token).
```
# my TOKEN is 40 characters - a robot account's password/"token" is 64 characters
export TOKEN="69JPGcPNONd0LEDXyPGDpMHkFwt6GHJRETjFdI7N"

# sad that this query doesn't list repos as "org/repo"
curl -s -X GET -H "Authorization: Bearer $TOKEN" 'https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/api/v1/repository?public=true' | jq --raw-output '.repositories[].name' 

# must delete "org/repo" instead of "repo"
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" 'https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/api/v1/repository/advanced-cluster-security/rhacs-operator-bundle'

# list all of the organizations, except the "adminorg" which holds my $TOKEN
curl -s -X GET -H "Authorization: Bearer $TOKEN" 'https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/api/v1/superuser/organizations/' | jq --raw-output '.organizations[].name' | grep -v adminorg

# deleting the org removes all repos
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/api/v1/superuser/organizations/advanced-cluster-security"

# delete all of the orgs -- DANGER!!!
for ORG in $(curl -s -X GET -H "Authorization: Bearer $TOKEN" 'https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/api/v1/superuser/organizations/' | jq --raw-output '.organizations[].name' | grep -v adminorg ); do
  echo "Deleting \"$ORG\" organization..."
  curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/api/v1/superuser/organizations/$ORG"
done