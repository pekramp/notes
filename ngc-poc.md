# Installing ngc-poc simulated ocp disconnected in aws step-by-step
Setup server
```
#register system
subscription-manager register
subscription-manager auto-attach

#update system
sudo dnf update

#grab oc
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/fast-4.12/openshift-client-linux-4.12.13.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/fast-4.12/openshift-install-linux-4.12.13.tar.gz

#Extract the client and place in into the path
sudo tar xzf openshift-client-linux-4.12.13.tar.gz -C /usr/local/sbin/ oc kubectl

sudo tar xzf openshift-install-linux-4.12.13.tar.gz -C /usr/local/sbin openshift-install

#set hostname to aws external dns name 
sudo  nmcli general hostname ec2-18-216-214-86.us-east-2.compute.amazonaws.com

```

Making an offline installation bundle for OpenShift requires [mirroring/downloading the container images](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html) and then [hosting those container images in a container registry](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-creating-registry.html) that is accessible by the cluster nodes. The download process can put the container images into the local filesystem or upload them directly into the container registry. (a USB stick, a directory that will be burnt to a DVD, or a folder that will be uploaded into S3 or similar storage)

:::info
A minimal download of OpenShift 4.12 requires ~15GB of space
A minimal download of OpenShift Platform Plus requires ~50GB of space
:::

Lots of good information in this blog - [Mirroring OpenShift Registries: The Easy Way by Ben Schmaus and Daniel Messer (August 23, 2022)](https://cloud.redhat.com/blog/mirroring-openshift-registries-the-easy-way).

## Install `mirror-registry` (aka mini Quay)
```
#pull down and extra mirror-registry
wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz

tar xvzf mirror-registry.tar.gz

./mirror-registry install --help


#setup
### password must be at least 8 characters and contain no whitespace
./mirror-registry install --quayRoot /data/mirror-registry --initUser admin --initPassword 3yHyHFb9ELEavGixZG846
```
:::info
INFO[2023-04-20 14:39:12] Quay installed successfully, config data is stored in ~/quay-install
INFO[2023-04-20 14:39:12] Quay is available at https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443 with credentials (admin, 3yHyHFb9ELEavGixZG846)
:::
```

#copy certificates to be trusted
sudo cp -v /data/mirror-registry/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

#optional cleanup if restarting
sudo rm /etc/pki/ca-trust/source/anchors/rootCA.pem
sudo update-ca-trust


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
oc-mirror list releases --version 4.12 --channels

oc-mirror list operators
oc-mirror list operators --version 4.12 --catalogs
oc-mirror list operators --version 4.12 \
    --catalog registry.redhat.io/redhat/redhat-operator-index:v4.12 \
    --package odf-operator \
    --channel stable-4.12


oc-mirror init | tee imageset-config.yaml
vi imageset-config.yaml

#find channels to slim down
podman pull registry.redhat.io/redhat/redhat-operator-index:v4.12
podman unshare
cd $(podman image mount registry.redhat.io/redhat/redhat-operator-index:v4.12)
ls configs

jq .name configs/*/catalog.json

# browse the metadata of an Operator
jq . configs/advanced-cluster-management/catalog.json | less -i

#list channels
jq '.schema, .name' configs/advanced-cluster-management/catalog.json
```
## Sample imageset-config.yaml
```
---
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: /data/oc-mirror-imageset-ngc
mirror:
  platform:
    channels:
    - name: fast-4.12
      type: ocp
      minVersion: 4.12.12
      maxVersion: 4.12.13
      shortestPath: true
    graph: true
  additionalImages:
  - name: registry.redhat.io/ubi8/ubi:latest
  - name: registry.redhat.io/openshift4/ose-cli:latest
  - name: docker.io/frekele/ant:latest
  - name: docker.io/library/amazoncorretto:latest
  helm: {}
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.12
    packages:
    - name: serverless-operator
      channels:
      - name: stable
    - name: ansible-automation-platform-operator
    - name: container-security-operator
      channels:
      - name: stable-3.8
    - name: cluster-logging
      channels:
      - name: stable
    - name: mcg-operator
      channels:
      - name: stable-4.12
    - name: mta-operator
      channels:
      - name: stable-v6.0
    - name: mtc-operator
      channels:
      - name: release-v1.7
    - name: mtr-operator
    - name: odf-operator
      channels:
      - name: stable-4.12
    - name: ocs-operator
      channels:
      - name: stable-4.12
    - name: odf-csi-addons-operator
      channels:
      - name: stable-4.12
    - name: odf-multicluster-orchestrator
      channels:
      - name: stable-4.12
    - name: local-storage-operator
    - name: odr-hub-operator
      channels:
      - name: stable-4.12
    - name: odr-cluster-operator
      channels:
      - name: stable-4.12
    - name: lvms-operator
      channels:
      - name: stable-4.12
    - name: elasticsearch-operator
      channels:
      - name: stable
    - name: rhsso-operator
      channels:
      - name: stable
    - name: advanced-cluster-management
      channels:
      - name: release-2.7
    - name: cincinnati-operator
      channels:
      - name: v1
    - name: compliance-operator
      channels:
      - name: stable
    - name: quay-bridge-operator
      channels:
      - name: stable-3.8
    - name: quay-operator
      channels:
      - name: stable-3.8
    - name: redhat-oadp-operator
      channels:
      - name: stable-1.1
    - name: rhacs-operator
      channels:
      - name: latest
```

## Run the oc-mirror
```
#prerun 
#df -h
#/ avail 88G

#fill the registry
time oc mirror --config=imageset-config.yaml docker://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/ngc-mirror

#time 
#info: Mirroring completed in 19m53.06s (56.23MB/s)
#Rendering catalog image "ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/ngc-mirror/redhat/redhat-operator-index:v4.12" with file-based catalog
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
## Get the mirror info for install
```
#cat mirror file
cat oc-mirror-workspace/results-1682046532/imageContentSourcePolicy.yaml
---
- mirrors:
  - ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/ngc-mirror/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/ngc-mirror/openshift/release-images
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

## Sample install-config
```
---
additionalTrustBundlePolicy: Always
apiVersion: v1
baseDomain: redhatgov.io
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m6a.2xlarge
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      type: m6a.xlarge
  replicas: 3
metadata:
  creationTimestamp: null
  name: ocp412ngc
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
  aws:
    region: us-east-2
    userTags:
      adminContact: pkramp
publish: External
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
  - ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/ngc-mirror/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443/ngc-mirror/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release

```

## Backup install-config
:::info
****Copy to a backup directory before running****
:::

## Run install
```
#start
time openshift-install create cluster --dir aws-install --log-level=info 

#output
INFO Credentials loaded from the "default" profile in file "/home/ec2-user/.aws/credentials"
INFO Consuming Install Config from target directory
INFO Creating infrastructure resources...
INFO Waiting up to 20m0s (until 4:12PM) for the Kubernetes API at https://api.ocp412ngc.redhatgov.io:6443...
INFO API v1.25.8+27e744f up
INFO Waiting up to 30m0s (until 4:23PM) for bootstrapping to complete...
INFO Destroying the bootstrap resources...
INFO Waiting up to 40m0s (until 4:44PM) for the cluster at https://api.ocp412ngc.redhatgov.io:6443 to initialize...
INFO Checking to see if there is a route at openshift-console/console...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/ec2-user/aws-install/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp412ngc.redhatgov.io
INFO Login to the console with user: "kubeadmin", and password: "qrPRj-nWGSZ-zjeAU-qWQw9"
INFO Time elapsed: 29m1s

real    29m0.825s
user    0m42.379s
sys     0m2.654s


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
oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' >ngcpull.json

#Copy
cp ngcpull.json ngc-updated-pull.json

#If missing can add with 
oc registry login --registry="https://ec2-18-216-214-86.us-east-2.compute.amazonaws.com:8443" --auth-basic="admin:3yHyHFb9ELEavGixZG846" --to=ngc-updated-pull.json

#Replace
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=ngc-updated-pull.json

#delete catalog source
oc get catalogsource --all-namespaces
oc delete catalogsource redhat-operator-index -n openshift-marketplace

```

## Destroy cluster
```
#start
time openshift-install destroy cluster --dir aws-install --log-level=info 
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