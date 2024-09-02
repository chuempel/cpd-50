# Installing CP4D 5.0.1 and the IBM Data Product Hub (DPH) service on a single-node OpenShift 4.15 cluster (SNO)

# 1. Provisioning a OCP SNO 4.15 cluster
You would need at least 32 vCPU, 128 GB RAM and 300 GB disk for the OCP 4.15 SNO cluster.

# 2. Preparing the bastion host
In this step, you will prepare the bastion host that come with the SNO host, to serve as NFS server to host the NFS storage for the OCP SNO cluster, and you will setup the bastion as a client workstation for Cloud Pak for Data to be able to install CPD on the OCP SNO.

### Step 2.1 - Login to the bastion host via ssh.
You will find the details how to ssh into the bastion host from the reservation details web site from above und "Bastion SSH Connection".
The password for the ssh connection to the bastion host can be found under "Bastion Password".

For example:
```
Bastion Password:
xxxx

Bastion SSH Connection:
ssh admin@xxx.yyy.ibm.com -p 22
```

### Step 2.2 - Change to root user and create install directory
Once you are logged into the bastion host via ssh, change to root user and create the install directory.
```
sudo su -
mkdir install
cd install
```

### Step 2.3 - Install the screen utility
```
yum -y install screen
```

### Step 2.4 - Download the CPD 5.0.1 cpd-cli utility
Download cpd-cli utility from github.com.

```
wget https://github.com/IBM/cpd-cli/releases/download/v14.0.1/cpd-cli-linux-EE-14.0.1.tgz
tar -xzvf cpd-cli-linux-EE-14.0.1.tgz
mv cpd-cli-linux-EE-14.0.1*/* .
rm -rf cpd-cli-linux-EE-14.0.1*
rm -f cpd-cli-linux-EE-14.0.1.tgz
./cpd-cli version
```

### Step 2.5 - Define environment variables
- the API URL
- the kubeadmin password
- your [IBM entitlement API key for Cloud Pak for Data](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=information-obtaining-your-entitlement-api-key)
- the API host, extracted from the API URL via command 
```
export SNO_API_URL=<replace with the value of API URL from OCP cluster connection details and remove /-sign at the end of the URL if present>
export SNO_CLUSTER_ADMIN_PWD=<replace with the value of Cluster Admin Password from the TechZone Reservation details>
export SNO_IBM_ENTITLEMENT_KEY=<replace with the value of your IBM Entitlement API key>
export SNO_API_HOST=$(echo $SNO_API_URL | sed 's/https:\/\///g' | sed 's/:6443//g')
```

An example for these environment variables would look like this:
```
export SNO_API_URL=https://api.xxx.yyy.ibm.com:6443
export SNO_CLUSTER_ADMIN_PWD=ByrhK-gkDbd-NIimN-xxxxx
export SNO_IBM_ENTITLEMENT_KEY=eyJ0eXAiOiJKV1QiLCxxxxx
export SNO_API_HOST=$(echo $SNO_API_URL | sed 's/https:\/\///g' | sed 's/:6443//g')
```

### Step 2.6 - Add an additional entry for the API host to the /etc/hosts file.
This is needed so that the cpd-cli manage commands from below, that will be passed to a container, can contact the OCP SNO cluster on the TechZone infrastruture.
```
echo "192.168.252.1 $SNO_API_HOST" >> /etc/hosts
```

Your updated /etc/hosts file should look like similar to this:
```
[root@bastion-gym-lan install]# more /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.252.1 api.xxx.yyy.ibm.com
```

### Step 2.7 - Validate that you can login as Kubeadmin user to your OCP SNO cluster
You should see a "Login successful" message.
```
oc login -u kubeadmin -p $SNO_CLUSTER_ADMIN_PWD $SNO_API_URL --insecure-skip-tls-verify
```

```
Login successful.

You have access to 70 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
Welcome! See 'oc help' to get started.
```


### Step 2.8 - Verify default storage class is managed-nfs-storage
```
oc get sc
```
You should see this output:
```
[root@bastion-gym-lan install]# oc get sc
NAME                            PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage (default)   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  4h46m
```


### Step 2.9 - Generate the cpd_vars.sh file
```
tee cpd_vars.sh <<EOF
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------
export OCP_URL="$SNO_API_HOST:6443"
export OPENSHIFT_TYPE="self-managed"
export OCP_USERNAME="kubeadmin"
export OCP_PASSWORD="$SNO_CLUSTER_ADMIN_PWD"
export OCP_TOKEN="$(oc whoami -t)"

# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------
export PROJECT_CERT_MANAGER="ibm-cert-manager"
export PROJECT_LICENSE_SERVICE="ibm-licensing"
export PROJECT_SCHEDULING_SERVICE="cpd-scheduler"
export PROJECT_CPD_INST_OPERATORS="cpd-operators"
export PROJECT_CPD_INST_OPERANDS="cpd-instance"

# ------------------------------------------------------------------------------
# Storage
# ------------------------------------------------------------------------------
export STG_CLASS_BLOCK=managed-nfs-storage
export STG_CLASS_FILE=managed-nfs-storage

# ------------------------------------------------------------------------------
# IBM Entitled Registry
# ------------------------------------------------------------------------------
export IBM_ENTITLEMENT_KEY=$SNO_IBM_ENTITLEMENT_KEY

# ------------------------------------------------------------------------------
# Cloud Pak for Data version
# ------------------------------------------------------------------------------
export VERSION=5.0.1

# ------------------------------------------------------------------------------
# Components
# ------------------------------------------------------------------------------
export COMPONENTS=dataproduct
EOF
```

### Step 2.10 - Verify your cpd_vars.sh file
```
cat cpd_vars.sh
```
Your generated cpd_vars.sh file should look similar to this:
```
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------
export OCP_URL="api.xxx.yyy.ibm.com:6443"
export OPENSHIFT_TYPE="self-managed"
export OCP_USERNAME="kubeadmin"
export OCP_PASSWORD="ByrhK-gkDbd-NIimN-krKUG"
export OCP_TOKEN="sha256~B10YB1V4bdcyzfz85u1EjUmxnQt5poq9gjJNhqCeNeQ"

# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------
export PROJECT_CERT_MANAGER="ibm-cert-manager"
export PROJECT_LICENSE_SERVICE="ibm-licensing"
export PROJECT_SCHEDULING_SERVICE="cpd-scheduler"
export PROJECT_CPD_INST_OPERATORS="cpd-operators"
export PROJECT_CPD_INST_OPERANDS="cpd-instance"

# ------------------------------------------------------------------------------
# NFS Server setup (optional)
# ------------------------------------------------------------------------------
export NFS_SERVER_LOCATION=<server-address>
export NFS_PATH=<path>
export PROJECT_NFS_PROVISIONER=nfs-provisioner
export NFS_STORAGE_CLASS=managed-nfs-storage
export NFS_IMAGE=registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2

# ------------------------------------------------------------------------------
# Storage
# ------------------------------------------------------------------------------
export STG_CLASS_BLOCK=managed-nfs-storage
export STG_CLASS_FILE=managed-nfs-storage

# ------------------------------------------------------------------------------
# IBM Entitled Registry
# ------------------------------------------------------------------------------
export IBM_ENTITLEMENT_KEY=eyJ0eXAiOiJKV1QiLCJxxx

# ------------------------------------------------------------------------------
# Cloud Pak for Data version
# ------------------------------------------------------------------------------
export VERSION=5.0.1

# ------------------------------------------------------------------------------
# Components
# ------------------------------------------------------------------------------
export COMPONENTS=dataproduct
```


# 3. Preparing the OpenShift cluster

### Step 3.1 - Source the CPD CLI environment variables from the cpd_vars.sh file.
```
source ./cpd_vars.sh
```

### Step 3.2 - Run the cpd-cli manage login command.
Note, that when running the command for the first time this will take some time to complete, as the olm-utils container image will be downloaded from the Internet.
```
./cpd-cli manage login-to-ocp --token=${OCP_TOKEN} --server=${OCP_URL}
```

Successful completion of the login command would look like the following.
```
[...]
Using project "default" on server "https://api.xxx.yyy.ibm.com:6443".
[SUCCESS] 2024-07-03T09:41:31.665896Z You may find output and logs in the /root/install/cpd-cli-workspace/olm-utils-workspace/work directory.
[SUCCESS] 2024-07-03T09:41:31.665924Z The login-to-ocp command ran successfully.
```

### Step 3.3.1 Change the Kubelet config (for non-Single-Node OpenShift clusters)
Run the cpd-cli manage apply-db2-kubelet command.
```
./cpd-cli manage apply-db2-kubelet
```

### Step 3.3.2 - Change the Kubelet config (only for Single-Node OpenShift)
In the Kubelet config, change CRI-O settings (pids limit), increase the maximal numbers of the pods on the SNO to 500 and enable unsafe systctls for the Db2 services.
```
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: cp4d-sno-config
spec:
  machineConfigPoolSelector:
    matchLabels:
      pools.operator.machineconfiguration.openshift.io/master: ""
  kubeletConfig:
    podPidsLimit: 12288
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "kernel.shm*"
      - "kernel.sem"
    podsPerCore: 0
    maxPods: 500
EOF
```

```
kubeletconfig.machineconfiguration.openshift.io/cp4d-sno-config created
```

### Step 3.4 - Watch the Kubelet changes being applied
Your OCP cluster will be rebooted to apply the changes. Verify that the changes have been applied.
```
watch -n 10 oc get mcp
```

Wait until the output of the command shows similar output to this. This could take up to 10 minutes.
```
Every 10.0s: oc get mcp                                                                                                            bastion-gym-lan: Wed Jul  3 09:38:03 2024

NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT
AGE
master   rendered-master-af82bf7616c60c2ba669f6e79a0d3354   True      False      False      1              1                   1                     0
5h31m
worker   rendered-worker-27aaddbc27c6cfe2d9d06101db8810a1   True      False      False      0              0                   0                     0
5h31m
```





### Step 3.5 - Setup NFS provisioner (optional)
```
oc new-project ${PROJECT_NFS_PROVISIONER}
```

```
./cpd-cli manage setup-nfs-provisioner \
--nfs_server=${NFS_SERVER_LOCATION} \
--nfs_path=${NFS_PATH} \
--nfs_provisioner_ns=${PROJECT_NFS_PROVISIONER} \
--nfs_storageclass_name=${NFS_STORAGE_CLASS} \
--nfs_provisioner_image=${NFS_IMAGE}
```


# 4. Installing CP4D 5.0.1 platform and DPH service

### Step 4.1 - Open a screen session.
This will allow your to reconnect to your terminal via screen -r if ever you loose the SSH connection to the bastion.
```
screen
```

### Step 4.2 - Create new projects for CPD the installation.
```
oc new-project ${PROJECT_CERT_MANAGER}
oc new-project ${PROJECT_LICENSE_SERVICE}
oc new-project ${PROJECT_SCHEDULING_SERVICE}
oc new-project ${PROJECT_CPD_INST_OPERATORS}
oc new-project ${PROJECT_CPD_INST_OPERANDS}
```

### Step 4.3 - Add your entitlement key to the global pull secret.
```
./cpd-cli manage add-icr-cred-to-global-pull-secret --entitled_registry_key=${IBM_ENTITLEMENT_KEY}
```

Output:
```
[SUCCESS] 2024-07-03T09:49:25.876626Z The add-icr-cred-to-global-pull-secret command ran successfully.
```

### Step 4.3.1 - Check connectivity to IBM Container registry
```
./cpd-cli manage login-entitled-registry ${IBM_ENTITLEMENT_KEY}
```

### Step 4.4.1 - Install the shared cluster components
This will install the cert manager and licensing services.
```
./cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
--licensing_ns=${PROJECT_LICENSE_SERVICE} \
--case_download=true \
--from_oci=true
```

Output:
```
[SUCCESS] 2024-07-03T09:56:56.998900Z The apply-cluster-components command ran successfully.
```

### Step 4.4.2 - Verify the shared cluster components
```
oc get pods -n ${PROJECT_LICENSE_SERVICE}; oc get pods -n ${PROJECT_CERT_MANAGER}
```

Output:
```
NAME                                                              READY   STATUS      RESTARTS   AGE
f3268163dfece62acfe24169b11923c516ac44c33b0c55f15623a1cc60z7jf6   0/1     Completed   0          7m49s
ibm-licensing-catalog-n68pv                                       1/1     Running     0          10m
ibm-licensing-operator-6b8d675c98-9tvkf                           1/1     Running     0          7m33s
ibm-licensing-service-instance-6f54c846c7-ndtrb                   1/1     Running     0          3m52s
NAME                                                              READY   STATUS      RESTARTS   AGE
4fb72fb19f187e74f231228d92bd2793deba1e5b8a4beb1343239327abt72v5   0/1     Completed   0          8m26s
cert-manager-cainjector-cb68d96d7-66r6m                           1/1     Running     0          8m3s
cert-manager-controller-86d9857d98-kmnnk                          1/1     Running     0          8m3s
cert-manager-webhook-69cc487dfb-48d5l                             1/1     Running     0          8m2s
ibm-cert-manager-catalog-h6455                                    1/1     Running     0          11m
ibm-cert-manager-operator-7cf5557575-t5647                        1/1     Running     0          8m13s
```

### Step 4.5 - Authorize the instance topology
```
./cpd-cli manage authorize-instance-topology \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Output:
```
[SUCCESS] 2024-07-03T10:07:36.270414Z The authorize-instance-topology command ran successfully.
```

### Step 4.6.1 - Setup the instance topology
```
./cpd-cli manage setup-instance-topology \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--license_acceptance=true \
--case_download=true \
--from_oci=true
```

Output:
```
[SUCCESS] 2024-07-03T10:21:06.508595Z The setup-instance-topology command ran successfully.
```

### Step 4.6.2 - Verify that all the pods in cpd-operators namespace are running.
```
oc get pods -n ${PROJECT_CPD_INST_OPERATORS}
```

Output:
```
NAME                                                              READY   STATUS      RESTARTS   AGE
032a79062c2bd3d11b45769f9ae13499e7ee7597ba5aacc2b5501a9be02mqbp   0/1     Completed   0          4m8s
4129e6adddc171eb9dcd3d7ecf985deb788f42ecc05603c2d86d808038b9fxn   0/1     Completed   0          4m41s
8896088bd627bde2cbd2c690af4ba55a2dd11ac5d526dcf59dbec1a92986rsp   0/1     Completed   0          2m43s
cloud-native-postgresql-catalog-6zth5                             1/1     Running     0          6m41s
ibm-common-service-operator-7dd4454cd8-n2kmk                      1/1     Running     0          3m32s
ibm-cs-iam-catalog-hc5mq                                          1/1     Running     0          6m39s
ibm-cs-install-catalog-pr48z                                      1/1     Running     0          6m41s
ibm-events-operator-catalog-8wpbb                                 1/1     Running     0          6m39s
ibm-namespace-scope-operator-85799f779c-xqcq5                     1/1     Running     0          4m30s
opencloud-operators-pq4ql                                         1/1     Running     0          6m37s
operand-deployment-lifecycle-manager-85778b44f7-mhz4s             1/1     Running     0          33s
```

### Step 4.7 - Install the operators for cpd_platform and your CPD services
Run the apply-olm command to install the operators of cpd_platform service and all the CPD services that you have specified in the COMPONENTS environments variable.
```
./cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=cpd_platform,${COMPONENTS} \
--case_download=true \
--from_oci=true
```

Output:
```
[SUCCESS] 2024-07-04T07:08:18.907001Z The apply-olm command ran successfully.
```

Verify that all the pods in cpd-operators namespace are running. You will see that many more appear, as we have selected to install the DPH service.
```
oc get pods -n ${PROJECT_CPD_INST_OPERATORS}
```

Output:
```
4129e6adddc171eb9dcd3d7ecf985deb788f42ecc05603c2d86d80803828524   0/1     Completed   0          31m
417890f798f7c1e64b68d3f4d68113e9696475849740350a415fdc9c55rr9nf   0/1     Completed   0          18m
5913e5ba5464c1058cb60aa3e3cc9bd88c0ea7e3d3e44ab2be07ca6a35m9j5r   0/1     Completed   0          21m
7b75fe5c6cf73ffc123909e6e54b5f2f299e071d750be693ffda4cc1ba8svcx   0/1     Completed   0          19m
8261ceabe37e521df5871078c190c7ca8afdcc0a7c543a91d203afd420prlcq   0/1     Completed   0          21m
8896088bd627bde2cbd2c690af4ba55a2dd11ac5d526dcf59dbec1a929qp6qr   0/1     Completed   0          29m
91561992bd053b3cddeeedf71a502f171901d854427d9b4208284125997npqm   0/1     Completed   0          17m
apple-fdb-controller-manager-565666ffcb-s29s4                     1/1     Running     0          17m
cloud-native-postgresql-catalog-lls4v                             1/1     Running     0          32m
cpd-platform-operator-manager-857647d948-nrp2d                    1/1     Running     0          21m
cpd-platform-scd7x                                                1/1     Running     0          25m
d9816001bba5e9486819ca2643f37d015d53b099627d2d11481e6bbea7lghzz   0/1     Completed   0          17m
db2u-day2-ops-controller-manager-68989b9ffd-qp2qg                 1/1     Running     0          20m
db2u-operator-manager-56dff49b6b-rdh6d                            1/1     Running     0          20m
e0c11c1f91e7b3cee0a97579a3015f887310557179cd8bd728b8261643bv8t6   0/1     Completed   0          20m
f0a948ed4a31cf431d31725e3fa7f3f408399b12d663f1a3e3a68dd3e8dbpj4   0/1     Completed   0          21m
ibm-common-service-operator-6c59586598-lnkqs                      1/1     Running     0          30m
ibm-cpd-ae-operator-catalog-5d7sh                                 1/1     Running     0          24m
ibm-cpd-ae-operator-f4dc9575-54h7z                                1/1     Running     0          19m
ibm-cpd-ccs-operator-b964f595-zzhs2                               1/1     Running     0          21m
ibm-cpd-ccs-operator-catalog-hzw7h                                1/1     Running     0          24m
ibm-cpd-datarefinery-operator-6f6b57f7fd-sfclg                    1/1     Running     0          19m
ibm-cpd-datarefinery-operator-catalog-mdwbd                       1/1     Running     0          24m
ibm-cpd-datastage-operator-99d9c6f-2jjpb                          1/1     Running     0          17m
ibm-cpd-datastage-operator-catalog-fpr66                          1/1     Running     0          24m
ibm-cpd-wkc-operator-59df88bcb8-rrxqw                             1/1     Running     0          16m
ibm-cpd-wkc-operator-catalog-df6f2                                1/1     Running     0          24m
ibm-cs-iam-catalog-qmh98                                          1/1     Running     0          32m
ibm-cs-install-catalog-qqd4p                                      1/1     Running     0          32m
ibm-db2aaservice-cp4d-operator-catalog-xc5v5                      1/1     Running     0          24m
ibm-db2aaservice-cp4d-operator-controller-manager-77bf4478mlsts   1/1     Running     0          18m
ibm-db2uoperator-catalog-pftsq                                    1/1     Running     0          24m
ibm-elasticsearch-catalog-mjrtk                                   1/1     Running     0          24m
ibm-elasticsearch-operator-ibm-es-controller-manager-59746tnhxp   1/1     Running     0          20m
ibm-events-operator-catalog-llh4j                                 1/1     Running     0          32m
ibm-fdb-controller-manager-68cc4f5694-jt7lw                       1/1     Running     0          17m
ibm-fdb-operator-catalog-v62wn                                    1/1     Running     0          24m
ibm-namespace-scope-operator-85799f779c-6x7rh                     1/1     Running     0          30m
ibm-zen-operator-catalog-w8kx6                                    1/1     Running     0          25m
opencloud-operators-wbzc6                                         1/1     Running     0          32m
operand-deployment-lifecycle-manager-85778b44f7-9hkvg             1/1     Running     0          27m
```

### Step 4.8 - Install the CPD platform by running the apply-cr command.
Note, that this command eventually takes 2 hours to complete due to the large number of services that will be installed when installing DPH.
```
./cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=cpd_platform \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
-v
```

Output:
```
[SUCCESS] 2024-07-04T08:22:04.205236Z The apply-cr command ran successfully.
```

Verify pods in cpd-instance namespace.
```
oc get pods -n ${PROJECT_CPD_INST_OPERANDS}
```

Output:
```
NAME                                                 READY   STATUS      RESTARTS   AGE
common-service-db-1                                  1/1     Running     0          7h26m
common-service-db-2                                  1/1     Running     0          7h25m
common-web-ui-76d85f86d9-vv467                       1/1     Running     0          7h19m
create-secrets-job-45phd                             0/1     Completed   0          7h14m
diagnostics-cronjob-28668650-f4lqk                   0/1     Completed   0          9m9s
iam-config-job-qvjsj                                 0/1     Completed   0          7h1m
ibm-mcs-hubwork-8b8b8bf47-qqbw6                      1/1     Running     0          6m28s
ibm-mcs-placement-6c7bcf79fb-btplm                   1/1     Running     0          5m27s
ibm-mcs-storage-744fc6d88c-wdhn5                     3/3     Running     0          2m7s
ibm-nginx-8bfcd6bbd-bmtkj                            2/2     Running     0          7h1m
ibm-nginx-8bfcd6bbd-vmdgg                            2/2     Running     0          7h1m
ibm-nginx-tester-55964bbddb-sb8dl                    2/2     Running     0          7h1m
ibm-zen-vault-sdk-jwt-setup-job-fb6ng                0/1     Completed   0          7h9m
oidc-client-registration-x4dgl                       0/1     Completed   0          7h29m
platform-auth-service-69666685b-c24d7                1/1     Running     0          7h25m
platform-identity-management-5b8c55cbb6-sxnjm        1/1     Running     0          7h25m
platform-identity-provider-574f6547db-rr97p          1/1     Running     0          7h25m
usermgmt-8c555755c-9vg2p                             1/1     Running     0          7h
usermgmt-8c555755c-ts8p2                             1/1     Running     0          7h
usermgmt-ensure-tables-job-k6rvs                     0/1     Completed   0          7h10m
watchdog-alert-monitoring-cronjob-28668650-pbztq     0/1     Completed   0          9m9s
zen-audit-65ff8cf949-hblcr                           1/1     Running     0          7h4m
zen-core-7c7f5f4f57-j67tr                            2/2     Running     0          7h4m
zen-core-7c7f5f4f57-zks5n                            2/2     Running     0          7h4m
zen-core-api-5f754775c4-z726z                        2/2     Running     0          7h4m
zen-core-api-5f754775c4-zk2n2                        2/2     Running     0          7h4m
zen-core-create-tables-job-95nms                     0/1     Completed   0          7h9m
zen-core-pre-requisite-job-n2m4w                     0/1     Completed   0          7h5m
zen-metastore-backup-cron-job-28668480-d5bq9         0/1     Completed   0          179m
zen-metastore-edb-1                                  1/1     Running     0          7h11m
zen-metastore-edb-2                                  1/1     Running     0          7h11m
zen-minio-0                                          1/1     Running     0          7h15m
zen-minio-1                                          1/1     Running     0          7h15m
zen-minio-2                                          1/1     Running     0          7h15m
zen-minio-create-buckets-job-hg65n                   0/1     Completed   0          7h14m
zen-pre-requisite-job-922qm                          0/1     Completed   0          7h6m
zen-remote-svc-inst-status-cron-job-28668642-97fd9   0/1     Completed   0          17m
zen-remote-svc-inst-status-cron-job-28668648-d65ws   0/1     Completed   0          11m
zen-remote-svc-inst-status-cron-job-28668654-gfbq2   0/1     Completed   0          5m9s
zen-svc-inst-status-cron-job-28668648-jmpx2          0/1     Completed   0          11m
zen-svc-inst-status-cron-job-28668652-7k597          0/1     Completed   0          7m9s
zen-svc-inst-status-cron-job-28668656-ppkvw          0/1     Completed   0          3m9s
zen-watchdog-55cff6d549-mkg87                        1/1     Running     0          6h56m
zen-watchdog-create-tables-job-ql78h                 0/1     Completed   0          7h9m
zen-watchdog-post-requisite-job-zh5tf                0/1     Completed   0          6h55m
zen-watchdog-pre-requisite-job-llzt7                 0/1     Completed   0          6h56m
zen-watcher-7984fd87d4-zmq2j                         2/2     Running     0          7h4m
```

### Step 4.9 - Install the DPH service by running the apply-cr command.
Note, that this command eventually takes 2 hours to complete due to the large number of services that will be installed when installing DPH.
```
./cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=${COMPONENTS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
-v
```

Output:
```
TASK [utils : check if CR status indicates completion for wkc-cr in cpd-instance, max retry 300 times 60s delay] ***********************************************************
Not ready yet - Retrying: check if CR status indicates completion for wkc-cr in cpd-instance, max retry 300 times 60s delay (300 Retries left)
Not ready yet - Retrying: check if CR status indicates completion for wkc-cr in cpd-instance, max retry 300 times 60s delay (299 Retries left)
Not ready yet - Retrying: check if CR status indicates completion for wkc-cr in cpd-instance, max retry 300 times 60s delay (298 Retries left)
```

Open a second SSH session to watch the installation progressing. You will need to wait 5-10 minuted until the first pods will startup in the CPD instance namespace, that this command monitors:
```
watch -n 10 "oc get pods -n ${PROJECT_CPD_INST_OPERANDS} --sort-by=.status.startTime | tac"
```
Output:
```
Every 10.0s: oc get pods -n cpd-instance --sort-by=.status.startTime | tac 

watchdog-alert-monitoring-cronjob-28668670-ttlcs                  0/1     ContainerCreating   0          9s
diagnostics-cronjob-28668670-sbgnb                                0/1     ContainerCreating   0          9s
zen-watchdog-cronjob-28668670-zwnbs                               0/1     ContainerCreating   0          9s
spark-hb-load-postgres-db-specs-5gvzd                             0/1     ContainerCreating   0          21s
zen-svc-inst-status-cron-job-28668668-8v8pk                       0/1     Completed           0          2m9s
zen-database-core-5c9758848d-99f9m                                1/1     Running             0          2m24s
elasticsea-0ac3-ib-6fb9-es-server-esnodes-2                       2/2     Running             0          2m26s
elasticsea-0ac3-ib-6fb9-es-server-esnodes-1                       2/2     Running             0          2m28s
elasticsea-0ac3-ib-6fb9-es-server-esnodes-0                       2/2     Running             0          2m29s
rabbitmq-ha-2                                                     1/1     Running             0          3m12s
spark-hb-create-trust-store-587866f4b9-lg976                      1/1     Running             0          3m24s
elasticsearch-master-ibm-elasticsearch-create-snapshot-repsdgsl   0/1     Completed           0          3m34s
zen-remote-svc-inst-status-cron-job-28668666-279xn                0/1     Completed           0          4m9s
rabbitmq-ha-1                                                     1/1     Running             0          4m14s
redis-ha-server-2                                                 2/2     Running             0          4m16s
spark-hb-cloud-native-postgresql-2                                1/1     Running             0          4m18s
redis-ha-server-1                                                 2/2     Running             0          4m39s
spark-hb-cloud-native-postgresql-1                                1/1     Running             0          4m51s
redis-ha-haproxy-cc768c86c-sdkdg                                  1/1     Running             0          5m
redis-ha-server-0                                                 2/2     Running             0          5m6s
rabbitmq-ha-0                                                     1/1     Running             0          5m26s
wdp-couchdb-0                                                     2/2     Running             0          5m42s
wdp-couchdb-1                                                     2/2     Running             0          5m42s
wdp-couchdb-2                                                     2/2     Running             0          5m42s
dsx-requisite-pre-install-job-tjck7                               0/1     Completed           0          6m5s
[...]
```

Output (after DPH installation completed):
```
[SUCCESS] 2024-07-04T16:15:13.927437Z The apply-cr command ran successfully.
```

Footprint after DPH installation:
```
oc describe nodes | grep -e "cpu  " -e "memory  "
```
```
  cpu                21746m (34%)   115850m (182%)
  memory             76087Mi (29%)  250696Mi (97%)
```

Ouput of pods in cpd instance namespace:
```
oc get pods -n ${PROJECT_CPD_INST_OPERANDS}
```

```
NAME                                                              READY   STATUS      RESTARTS   AGE
asset-files-api-847557985-sds7x                                   1/1     Running     0          60m
ax-environments-api-deploy-5d6fd7544c-strws                       1/1     Running     0          54m
ax-environments-ui-deploy-7b8cb98567-gfnrw                        1/1     Running     0          54m
c-db2oltp-wkc-db2u-0                                              1/1     Running     0          56m
c-db2oltp-wkc-instdb-hx4cs                                        0/1     Completed   0          59m
catalog-api-856ccc96c5-lqsh5                                      1/1     Running     0          68m
catalog-api-856ccc96c5-t2gg6                                      1/1     Running     0          68m
ccs-post-install-job-p9h9m                                        0/1     Completed   0          44m
common-service-db-1                                               1/1     Running     0          8h
common-service-db-2                                               1/1     Running     0          8h
common-web-ui-76d85f86d9-vv467                                    1/1     Running     0          8h
conn-home-pre-install-job-v48wr                                   0/1     Completed   0          68m
connect-sdk-init-h44l5                                            0/1     Completed   0          69m
copy-scheduler-runtime-def-job-dnjx2                              0/1     Completed   0          51m
create-dap-directories-m4mmr                                      0/1     Completed   0          63m
create-secrets-job-45phd                                          0/1     Completed   0          8h
dap-base-extension-translations-jjlts                             0/1     Completed   0          61m
dap-base-folder-global-type-5r774                                 0/1     Completed   0          61m
dataview-api-service-5cf7445d9f-dtkwg                             1/1     Running     0          50m
dc-main-7db8cbc5c4-hj2t4                                          1/1     Running     0          66m
diagnostics-cronjob-28668730-drpgf                                0/1     Completed   0          9m21s
dp-transform-564cb485c-7mh8w                                      1/1     Running     0          17m
dp-transform-iae-thirdparty-lib-vol-pt8pq                         0/1     Completed   0          19m
dsx-requisite-pre-install-job-tjck7                               0/1     Completed   0          75m
elasticsea-0ac3-ib-6fb9-es-server-esnodes-0                       2/2     Running     0          71m
elasticsea-0ac3-ib-6fb9-es-server-esnodes-1                       2/2     Running     0          71m
elasticsea-0ac3-ib-6fb9-es-server-esnodes-2                       2/2     Running     0          71m
elasticsearch-master-ibm-elasticsearch-create-snapshot-repsdgsl   0/1     Completed   0          72m
env-spec-sync-job-28668735-jdzwk                                  0/1     Completed   0          4m21s
environments-init-job-9ksj4                                       0/1     Completed   0          55m
event-logger-api-c59c45bb8-ts5th                                  1/1     Running     0          60m
finley-public-9c499b7f-hlqgl                                      1/1     Running     0          26m
iam-config-job-qvjsj                                              0/1     Completed   0          8h
ibm-mcs-hubwork-8b8b8bf47-qqbw6                                   1/1     Running     0          86m
ibm-mcs-placement-6c7bcf79fb-btplm                                1/1     Running     0          85m
ibm-mcs-storage-744fc6d88c-wdhn5                                  3/3     Running     0          82m
ibm-nginx-8bfcd6bbd-bmtkj                                         2/2     Running     0          8h
ibm-nginx-8bfcd6bbd-vmdgg                                         2/2     Running     0          8h
ibm-nginx-tester-55964bbddb-sb8dl                                 2/2     Running     0          8h
ibm-zen-vault-sdk-jwt-setup-job-fb6ng                             0/1     Completed   0          8h
jobs-api-6d88849fb6-6jpsj                                         1/1     Running     0          52m
jobs-ui-8456dd9f75-5f82h                                          1/1     Running     0          52m
jobs-ui-extension-translations-n5flg                              0/1     Completed   0          52m
knowledge-accelerators-8547477888-sjdnn                           1/1     Running     0          17m
metadata-discovery-5bbfd6bf86-j84rf                               1/1     Running     0          17m
ngp-projects-api-7d4cc49d55-x8nwn                                 1/1     Running     0          60m
oidc-client-registration-x4dgl                                    0/1     Completed   0          8h
platform-auth-service-69666685b-c24d7                             1/1     Running     0          8h
platform-identity-management-5b8c55cbb6-sxnjm                     1/1     Running     0          8h
platform-identity-provider-574f6547db-rr97p                       1/1     Running     0          8h
portal-catalog-5dcd58f444-9f84s                                   1/1     Running     0          66m
portal-common-api-79d45cc868-7d6ds                                1/1     Running     0          23m
portal-job-manager-7ff98c476b-2lp6n                               1/1     Running     0          60m
portal-main-6fbb5d95b4-7zhlx                                      1/1     Running     0          61m
portal-notifications-7c5567c8f7-gfqbd                             1/1     Running     0          60m
portal-projects-697c4d5f88-7gqmv                                  1/1     Running     0          61m
post-install-upgrade-spaces-wml-global-asset-type-mhmw5           0/1     Completed   0          45m
projects-ui-refresh-users-v87kf                                   0/1     Completed   0          44m
rabbitmq-ha-0                                                     1/1     Running     0          74m
rabbitmq-ha-1                                                     1/1     Running     0          73m
rabbitmq-ha-2                                                     1/1     Running     0          72m
redis-ha-haproxy-cc768c86c-sdkdg                                  1/1     Running     0          74m
redis-ha-server-0                                                 2/2     Running     0          74m
redis-ha-server-1                                                 2/2     Running     0          73m
redis-ha-server-2                                                 2/2     Running     0          73m
runtime-assemblies-operator-7d5ff85cb8-sldhq                      1/1     Running     0          59m
runtime-manager-api-69446b47d-qphng                               1/1     Running     0          59m
scheduler-rtm-upgrade-job-xdtz7                                   0/1     Completed   0          43m
spaces-64fc5684b4-kzcsm                                           1/1     Running     0          51m
spaces-ui-extension-translations-vvbsw                            0/1     Completed   0          51m
spark-hb-br-recovery-6775f89cf8-8lqkp                             1/1     Running     0          63m
spark-hb-cloud-native-postgresql-1                                1/1     Running     0          74m
spark-hb-cloud-native-postgresql-2                                1/1     Running     0          73m
spark-hb-control-plane-859ccd577f-t7rkn                           2/2     Running     0          67m
spark-hb-create-trust-store-587866f4b9-lg976                      1/1     Running     0          72m
spark-hb-deployer-agent-59bdcf9b57-jp47q                          2/2     Running     0          68m
spark-hb-job-cleanup-cron-28668720-pdk9n                          0/1     Completed   0          19m
spark-hb-kernel-cleanup-cron-28668720-rrt2j                       0/1     Completed   0          19m
spark-hb-load-postgres-db-specs-5gvzd                             0/1     Completed   0          69m
spark-hb-nginx-646778f75b-99l22                                   1/1     Running     0          67m
spark-hb-preload-jkg-image-28668720-4cdc9                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-6fcs4                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-6nk8h                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-7g4fc                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-86qgd                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-8wkl2                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-b842k                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-dqh7v                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-fzsss                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-g4vtf                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-gtj7b                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-krxh2                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-l2ldj                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-l5h9m                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-mwf4s                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-s8rqf                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-sgpjs                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-wbnpr                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-zl6xg                         0/2     Completed   0          19m
spark-hb-preload-jkg-image-28668720-zpl8m                         0/2     Completed   0          19m
spark-hb-register-hb-dataplane-5b6579d4f6-jxw8m                   1/1     Running     0          53m
spark-hb-ui-6845978496-kxpvh                                      1/1     Running     0          67m
task-credentials-78cd458c79-2pfbl                                 1/1     Running     0          51m
usermgmt-8c555755c-9vg2p                                          1/1     Running     0          8h
usermgmt-8c555755c-ts8p2                                          1/1     Running     0          8h
usermgmt-ensure-tables-job-k6rvs                                  0/1     Completed   0          8h
volumes-iaewdpthirdpartylib-deploy-859d896687-n7rxm               1/1     Running     0          18m
volumes-iaewdpthirdpartylib-start-file-server-job-t9fwm           0/1     Completed   0          18m
volumes-profstgintrnl-deploy-856dd64dd7-msbmz                     1/1     Running     0          21m
volumes-profstgintrnl-start-file-server-job-6sg8b                 0/1     Completed   0          21m
watchdog-alert-monitoring-cronjob-28668730-zrht9                  0/1     Completed   0          9m21s
wdp-connect-connection-57bb44ddbd-jvhtk                           1/1     Running     0          67m
wdp-connect-connector-567779fc9f-dsk9x                            1/1     Running     0          67m
wdp-connect-flight-6f9898cb8f-mrfxt                               1/1     Running     0          67m
wdp-couchdb-0                                                     2/2     Running     0          74m
wdp-couchdb-1                                                     2/2     Running     0          74m
wdp-couchdb-2                                                     2/2     Running     0          74m
wdp-dataprep-6bf7f49f54-x7snh                                     1/1     Running     0          35m
wdp-dataview-54c5866787-r9fhd                                     1/1     Running     0          50m
wdp-lineage-7d4fc4fdb9-rqcxd                                      1/1     Running     0          20m
wdp-policy-service-9dcdc5d4b-q6zpk                                1/1     Running     0          21m
wdp-profiling-5897c5d6f5-hvswl                                    1/1     Running     0          16m
wdp-profiling-iae-init-bwtj8                                      0/1     Completed   0          21m
wdp-profiling-iae-thirdparty-lib-vol-xtqwc                        0/1     Completed   0          17m
wdp-profiling-iae-thirdparty-lib-volume-instance-jzvqx            0/1     Completed   0          19m
wdp-profiling-messaging-d886bc9ff-c9wml                           1/1     Running     0          16m
wdp-profiling-ui-665cc765bc-6cpl5                                 1/1     Running     0          15m
wdp-search-cc964c767-qmj2w                                        1/1     Running     0          20m
wdp-shaper-5c4bfcdf89-bbs7n                                       1/1     Running     0          35m
wkc-bi-data-service-5f6d6759db-nr7pv                              1/1     Running     0          19m
wkc-catalog-api-jobs-7587bdc6bd-cbnm2                             1/1     Running     0          17m
wkc-data-rules-6944bd95d9-zflf9                                   1/1     Running     0          20m
wkc-db2u-init-g2q5d                                               0/1     Completed   0          20m
wkc-extensions-translations-init-t7cbt                            0/1     Completed   0          66m
wkc-glossary-service-f8db75fdb-dmjt7                              1/1     Running     0          23m
wkc-glossary-service-sync-cronjob-28668721-6c52v                  0/1     Completed   0          18m
wkc-gov-ui-69849bc7f5-j5drd                                       1/1     Running     0          23m
wkc-mde-service-manager-69886d86bd-sjp4f                          1/1     Running     0          16m
wkc-metadata-imports-ui-78f6574668-fkxgh                          1/1     Running     0          17m
wkc-post-install-init-mnwdr                                       0/1     Completed   0          15m
wkc-roles-init-jnjtf                                              0/1     Completed   0          21m
wkc-search-678767465d-5d8s4                                       1/1     Running     0          66m
wkc-term-assignment-569855cdd6-s5lws                              1/1     Running     0          16m
wkc-workflow-service-55f4dd7b68-tplcm                             1/1     Running     0          23m
wml-main-7bdb79c89-kfj2x                                          1/1     Running     0          51m
zen-audit-65ff8cf949-hblcr                                        1/1     Running     0          8h
zen-core-7c7f5f4f57-j67tr                                         2/2     Running     0          8h
zen-core-7c7f5f4f57-zks5n                                         2/2     Running     0          8h
zen-core-api-5f754775c4-z726z                                     2/2     Running     0          8h
zen-core-api-5f754775c4-zk2n2                                     2/2     Running     0          8h
zen-core-create-tables-job-95nms                                  0/1     Completed   0          8h
zen-core-pre-requisite-job-n2m4w                                  0/1     Completed   0          8h
zen-database-core-5c9758848d-99f9m                                1/1     Running     0          71m
zen-metastore-backup-cron-job-28668480-d5bq9                      0/1     Completed   0          4h19m
zen-metastore-edb-1                                               1/1     Running     0          8h
zen-metastore-edb-2                                               1/1     Running     0          8h
zen-minio-0                                                       1/1     Running     0          8h
zen-minio-1                                                       1/1     Running     0          8h
zen-minio-2                                                       1/1     Running     0          8h
zen-minio-create-buckets-job-hg65n                                0/1     Completed   0          8h
zen-pre-requisite-job-922qm                                       0/1     Completed   0          8h
zen-remote-svc-inst-status-cron-job-28668726-gg6qx                0/1     Completed   0          13m
zen-remote-svc-inst-status-cron-job-28668732-c2xh9                0/1     Completed   0          7m21s
zen-remote-svc-inst-status-cron-job-28668738-dfnwr                0/1     Completed   0          81s
zen-svc-inst-status-cron-job-28668728-spx7d                       0/1     Completed   0          11m
zen-svc-inst-status-cron-job-28668732-c2vts                       0/1     Completed   0          7m21s
zen-svc-inst-status-cron-job-28668736-kdvsx                       0/1     Completed   0          3m21s
zen-watchdog-55cff6d549-mkg87                                     1/1     Running     0          8h
zen-watchdog-create-tables-job-ql78h                              0/1     Completed   0          8h
zen-watchdog-post-requisite-job-zh5tf                             0/1     Completed   0          8h
zen-watchdog-pre-requisite-job-llzt7                              0/1     Completed   0          8h
zen-watcher-7984fd87d4-5hqwp                                      2/2     Running     0          23m
```

# 5. Post-installation steps

### Step 5.1 - Retrieving the default CPD admin password
```
oc -n ${PROJECT_CPD_INST_OPERANDS} get secret ibm-iam-bindinfo-platform-auth-idp-credentials \
   -o 'jsonpath={.data.admin_password}'| base64 --decode ;echo
```

### Step 5.2 - Retrieving the CPD web console URL
```
oc get routes -A | grep ibm-nginx-svc | awk '{print "https://" $3}'
```

### Step 5.3 - Measuring the footprint
```
oc describe nodes | grep -e "cpu  " -e "memory  "
```

### Step 5.4 - Logging in to the SNO node
```
oc debug node/master-1
```

# 6. Troubleshooting
If you are experiencing problems with accessing the SNO cluster via oc login commands, for example getting an EOF error, it might help to reboot the SNO node.

### Step 6.1 - Rebooting the SNO worker node
You can ssh to the SNO node from your bastion and then rebooting the node.
```
ssh core@192.168.252.11 -i /tmp/ocp/cluster/id_rsa
sudo su -
shutdown -Fr now
```

# 7. Setting up Minio S3
This explains how to setup a Minio S3 cluster on the OCP cluster.

Assumes that:
- you are logged in as kubeadmin to your OpenShift cluster
- you can pull images from Dockerhub
- you have a storage class nfs-storage-provisioner

### Step 7.1 Sample Object Store using MinIO

For testing purposes, steps are shown to install a local MinIO server, which is an
open-source object store.

### Step 7.2 Example using NFS storage class:
1.  ```wget https://github.com/vmware-tanzu/velero/releases/download/v1.6.0/velero-v1.6.0-linux-amd64.tar.gz```
2.  ```tar xvfz velero-v1.6.0-linux-amd64.tar.gz```
3.  From the extracted velero folder, run the following. This creates a
    sample MinIO deployment in the "velero" namespace.
    ```
    oc apply -f examples/minio/00-minio-deployment.yaml
    ```
    MinIO pulls images from docker.io. docker.io is subject to rate limiting.
    
    minio pods may fail with error:
    ```
    toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
    ```

    If this occurs, try the following:

    1.  Create a docker account and login

    2.  Obtain an access token from
        <https://hub.docker.com/settings/security>

    3.  Create a docker pull secret.  Substitute 'myuser' and 'myaccesstoken' with the docker account user and token.
        ```
        oc create secret docker-registry --docker-server=docker.io --docker-username=myuser --docker-password=myaccesstoken -n velero dockerpullsecret
        ```
    4.  Add the image pull secret to the 'default' service account
        ```
        oc secrets link default dockerpullsecret --for=pull -n velero
        ```

    5.  Restart the minio pods

### Step 7.3 Update the image used by the minio pod
   ```
   oc set image deployment/minio minio=minio/minio:RELEASE.2021-04-22T15-44-28Z -n velero
   ```

### Step 7.4 Creating PVCs for MinIO
Create two persistent volumes and update the deployment. Change the storage class and size as needed.

Create config PVC
```
oc apply -f minio-config-pvc.yaml
```
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: velero
  name: minio-config-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
      storageClassName: managed-nfs-storage
```

Create storage PVC
```
oc apply -f minio-storage-pvc.yaml
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: velero
  name: minio-storage-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 400Gi
      storageClassName: managed-nfs-storage
```

Set config volume
```
oc set volume deployment.apps/minio --add --overwrite --name=config --mount-path=/config \
  --type=persistentVolumeClaim --claim-name="minio-config-pvc" -n velero
```
    
Set storage volume
```
oc set volume deployment.apps/minio --add --overwrite --name=storage --mount-path=/storage \
   --type=persistentVolumeClaim --claim-name="minio-storage-pvc" -n velero
```

### Step 7.5 Set resource limits for the minio deployment.
```
oc set resources deployment minio -n velero --requests=cpu=500m,memory=256Mi --limits=cpu=1,memory=1Gi
```

### Step 7.6 Check that the MinIO pods are up and running.
```
oc get pods -n velero
```

### Step 7.7 Expose the minio service
```
oc expose svc minio -n velero
```

### Step 7.8 Get the MinIO URL
```
oc get route minio -n velero
```
Example:
    http://minio-velero.apps.mycluster.cp.fyre.ibm.com
    User/password
    minio/minio123

### Step 7.9 Go to the MinIO web UI and create a bucket called "velero"

### Step 7.10 - Create Ingress for Minio
Create the following Ingress and save the YAML in the minio-ingress.yaml file
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio
spec:
  rules:
  - host: minio-velero.apps.cp4d.mop
    http:
      paths:
      - backend:
          # Forward to a Service called 'minio'
          service:
            name: minio
            port:
              number: 9000
        path: /
        pathType: Exact
 ```
 
 Apply the YAML to create the Ingress for Minio on your cluster.
 ```
 oc apply -f minio-ingress.yaml
 ```
  
# 8. Storage validation
Perform a health check on storage validation.

The cpd-cli health storage-validation command uses the Storage Validation tool for IBM Cloud Paks to validate storage on ReadWriteOnce and ReadWriteMany volumes on a Red Hat® OpenShift® cluster.

The storage-validation command performs the following health checks:
- Dynamic provisioning of a volume
- Mounting volume from a node
- Sequential read/write consistency from single and multiple nodes
- Parallel read/write consistency from single and multiple nodes
- Parallel read/write consistency across multiple threads
- File permissions on mounted volumes
- Accessibility based on POSIX compliance group ID permissions
- SubPath test for volumes
- File locking test

References:
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=health-storage-validation
- https://github.com/IBM/k8s-storage-tests

### 8.1 Run the storage validation command
```
cpd-cli health storage-validation \
--param param.yml
```
## 9. Configuring LDAP

### 9.1 Connecting to your identity provider
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.0.x?topic=users-connecting-your-identity-provider  

## 10. Setting up HTTP proxy

### 10.1 Install RSI webhook
```
./cpd-cli manage install-rsi \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

### 10.2 Enable RSI webhook
```
./cpd-cli manage enable-rsi \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

### 10.3 Check RSI webhook is running
```
oc get mutatingwebhookconfiguration -n ${PROJECT_CPD_INST_OPERANDS} | grep rsi-webhook-cfg
```
Output:
```
rsi-webhook-cfg-cpd-instance                 1          7m35s
```

### 10.4 Configure proxy
```
export PROXY_HOST=<ip of proxy host>
export PROXY_PORT=<port of proxy host>
# for example:
# export PROXY_HOST=1.2.3.4
# export PROXY_PORT=8080 
```

```
./cpd-cli manage create-proxy-config \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--proxy_host=${PROXY_HOST} \
--proxy_port=${PROXY_PORT}
```

# END OF DOCUMENT

