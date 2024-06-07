
# Installing Openshift UPI in a restricted/disconnected environment

We have to follow below workflow for a disconnected environment installation.

1. Configure a container registry and then sync the openshift installation images. In the illustration we have used the `quay` registry to sync the images. We need internet access on the quay registry node. If we do not have internet access on the registry node, then we can download the images by using `oc-mirror` on a system which has internet access and then push them to the registry node.

2. After configuring the private registry, rest of the procedure for installation is almost same as the normal UPI.

3. Install haproxy/dns/http on the bastion node. Internet access on the bastion node is not mandatory, as all the required images will be synced by bastion node.

4. Once installation is completed, disable the default content sources and configure the cluster to pull sources from the private registry.

5. Configuring OLM to pull operators from private registry.


Following are the node configuration details.

| Hostname | IP address     | Description                       | Hardware Requirement | 
| :-------- | :------- | :-------------------------------- | :------------------- |
| `quay-all.local.com`      | `192.168.1.201` | `Registry Node` | 8 GB RAM  - 60GB HDD - 4 CPU |
| `service.local.com`      | `192.168.1.210` | `Bastion/helper Node` | 16 GB RAM  - 60GB HDD - 4 CPU |
| `bootstrap.local.com`      | `192.168.1.211` | `Bootstrap Node` | 16 GB RAM  - 60GB HDD - 4 CPU |
| `master1.local.com`      | `192.168.1.212` | `Master Node` |16 GB RAM  - 60GB HDD - 4 CPU |
| `master2.local.com`      | `192.168.1.213` | `Master Node` |16 GB RAM  - 60GB HDD - 4 CPU |
| `master3.local.com`      | `192.168.1.214` | `Master Node` |16 GB RAM  - 60GB HDD - 4 CPU |



1. Set IP address and hostname. As, I already have IP configured so I am just doing the hostname configuration and making an entry in /etc/hosts


```bash
# hostnamectl set-hostname quay-all.local.com
# ip=`hostname -I|awk '{print $1}'`
# echo $ip `hostname` >> /etc/hosts
# cat /etc/hosts 
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.201 quay-all.local.com

```

2. Register the quay node with `Red Hat` and also login to registry.redhat.io


```bash
# subscription-manager register
```

2. Install following Packages if not installed
```bash
# yum update -y
# yum install podman

```

3. Login to the registry.redhat.io

```bash
$ sudo podman login  registry.redhat.io

```

4. Configure firewall

```bash
# firewall-cmd --permanent --add-port=80/tcp \
&& firewall-cmd --permanent --add-port=443/tcp \
&& firewall-cmd --permanent --add-port=5432/tcp \
&& firewall-cmd --permanent --add-port=5433/tcp \
&& firewall-cmd --permanent --add-port=6379/tcp \
&& firewall-cmd --reload
```

5. 

```bash
# hostnamectl set-hostname quay-all.example.com
# ip=`hostname -I|awk '{print $1}'`
# echo $ip `hostname` >> /etc/hosts
```

6. Configure Databse container
```bash
# mkdir -p $QUAY/postgres-quay
# setfacl -m u:26:-wx $QUAY/postgres-quay
# podman run -d --rm --name postgresql-quay \
-e POSTGRESQL_USER=quayuser \
-e POSTGRESQL_PASSWORD=quaypass \
-e POSTGRESQL_DATABASE=quay \
-e POSTGRESQL_ADMIN_PASSWORD=adminpass \
-p 5432:5432 \
-v $QUAY/postgres-quay:/var/lib/pgsql/data:Z \
registry.redhat.io/rhel8/postgresql-13:1-109
# podman exec -it postgresql-quay /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | psql -d quay -U postgres'
```

7. Configuring Redis container

```bash
# podman run -d --rm --name redis \
  -p 6379:6379 \
  -e REDIS_PASSWORD=strongpassword \
  registry.redhat.io/rhel8/redis-6:1-110
```

8. Deploy quay container

```bash
# mkdir $QUAY/config;cd $QUAY/config
# cat <<EOF > config.yaml
BUILDLOGS_REDIS:
    host: quay-all.local.com
    password: strongpassword
    port: 6379
CREATE_NAMESPACE_ON_PUSH: true
DATABASE_SECRET_KEY: a8c2744b-7004-4af2-bcee-e417e7bdd235
DB_URI: postgresql://quayuser:quaypass@quay-all.local.com:5432/quay
DISTRIBUTED_STORAGE_CONFIG:
    default:
        - LocalStorage
        - storage_path: /datastorage/registry
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []
DISTRIBUTED_STORAGE_PREFERENCE:
    - default
FEATURE_MAILING: false
SECRET_KEY: e9bd34f4-900c-436a-979e-7530e5d74ac8
SERVER_HOSTNAME: quay-all.local.com
SETUP_COMPLETE: true
USER_EVENTS_REDIS:
    host: quay-all.local.com
    password: strongpassword
    port: 6379
EOF
# ls -l $QUAY/config/config.yaml
```
9. Prepare the storage and deploy container.

```bash
# mkdir $QUAY/storage
# setfacl -m u:1001:-wx $QUAY/storage
# podman run -d --rm -p 80:8080 -p 443:8443  \
   --name=quay \
   -v $QUAY/config:/conf/stack:Z \
   -v $QUAY/storage:/datastorage:Z \
   registry.redhat.io/quay/quay-rhel8:v3.11.1
```

10. Try to login 
```bash
# podman login --tls-verify=false quay-all.local.com
```

After installing the cluster and disabling the default operator sources, following pods are running in openshift-marketplace CREATE_NAMESPACE_ON_PUSH

```bash
oc get pods -n openshift-marketplace
NAME                                    READY   STATUS    RESTARTS      AGE
marketplace-operator-5486b959b7-rcs66   1/1     Running   4 (26m ago)   20h

oc get imagecontentsourcepolicy
NAME           AGE
image-policy   20h

oc get catsrc -A
No resources found

 oc get catalogsource -n openshift-marketplace
No resources found in openshift-marketplace namespace.

 oc get packagemanifest -n openshift-marketplace
No resources found in openshift-marketplace namespace.
```

Creating the imagecontentsourcepolicy. 
```bash
cat  cat imageContentSourcePolicy.yaml 
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  labels:
    operators.openshift.org/catalog: "true"
  name: operator-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - quay-all.example.com/quayadmin/prometheus-operator
    source: quay.io/prometheus-operator
  - mirrors:
    - quay-all.example.com/quayadmin/openshift-community-operators
    source: quay.io/openshift-community-operators


oc create -f imageContentSourcePolicy.yaml


cat catalogSource-cs-community-operator-index.yaml 
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: cs-community-operator-index
  namespace: openshift-marketplace
spec:
  image: quay-all.example.com/quayadmin/redhat/community-operator-index:v4.15
  sourceType: grpc


oc create -f catalogSource-cs-community-operator-index.yaml

oc get catalogsource -n openshift-marketplace
NAME                          DISPLAY   TYPE   PUBLISHER   AGE
cs-community-operator-index             grpc               7s

oc get imageContentSourcePolicy
NAME           AGE
image-policy   21h
operator-0     5m22s

```

To Allow OCP to pull images from the OCP registry. Add the pull-secret 

```bash

oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' > pull-secret
oc registry login --registry="<registry>" --auth-basic="quayadmin:redhat@123" --to=pull-secret
```
Verify that mcp has been changed to updating. It may take 10-20 minutes to update, may vary from cluster to cluster.

```bash
oc get mcp
```

Now check on the nodes, if credentials for private registry have been added.

```bash
for node in $(oc get nodes -o jsonpath='{.items[*].metadata.name}'); do oc debug node/${node} -- chroot /host cat /var/lib/kubelet/config.json; done
```
## Authors

- [@kaybee-singh](https://www.github.com/kaybee-singh)

