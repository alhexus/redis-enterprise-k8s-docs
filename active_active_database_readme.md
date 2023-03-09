<!-- omit in toc -->
# K8s Redis Enterprise Active Active Database README

This document describes how to deploy an Active-Active database with Redis Enterprise for Kubernetes.

**Disclaimer:**
  This feature is in private preview and is not complete for production use.
  Importent - Please view the [support for only one Active-Active database with up-to three participating clusters across a set of participating clusters](#support-for-only-one-active-active-database-with-up-to-three-participating-clusters-across-a-set-of-participating-clusters) limitation.
  

**This document assumes the following:**
  - You have at least two running Redis Enterprise clusters that will be used as your participating clusters.
  - On each Kubernetes cluster you have Ingress controller or Openshift Routes deployed.
  - You have admin access level for each Kubernetes cluster.

## Table of contents
  * [Overview](#overview)
  * [Create a new Active-Active database](#create-a-new-active-active-database)
  * [Add participating cluster to an existing Active-Active database](#add-participating-cluster-to-an-existing-active-active-database)
  * [Remove participating cluster from an existing Active-Active database](#remove-participating-cluster-from-an-existing-active-active-database)
  * [Delete an existing Active-Active database](#delete-an-existing-active-active-database)
  * [Update existing participating cluster (RERC) details](#update-existing-participating-cluster-rerc-details)
  * [Test your Active-Active database](#test-your-active-active-database)
  * [Limitations](#limitations)

## Overview

This feature allows creating and managing Active-Active databases with Kubernetes custom resources.
The new custom resources for declaring Active-Active databases in Redis Enterprise are:
  - Redis Enterprise Remote Cluster (RERC) will contain the participating cluster configurations.
  - Redis Enterprise Active Active Database (REAADB) will contain a link to the RERC for each participating cluster, and provide the management path configurations and status.

For example, an Active-Active database with two participating clusters should have two Redis Enterprise Remote Cluster (RERC) custom resources and one Redis Enterprise Active Active Database (REAADB) custom resource on each participating cluster.

## Create a new Active-Active database

**For each participating cluster with a namespace that hosts the Redis Enterprise operator, follow the [Participating cluster preparation setup](#participating-cluster-preparation-setup)**  

### Participating cluster preparation setup

#### Part 1: First time REC preparation setup

Note:
  * This part is for new participating clusters that are not already configured to use the Redis Enterprise Operator active-active databases.

1. Apply the following CRDs:
```
  kubectl apply -f reaadb_crd.yaml
  kubectl apply -f rerc_crd.yaml
```

Note:
  * This step requires "cluster role" permissions for applying CRD resources.

2. Add the following environment variables to enable the active active and remote cluster controllers on the Redis Enterprise operator Configmap:
```
  kubectl patch cm  operator-environment-config --type merge --patch "{\"data\": \
    {\"ACTIVE_ACTIVE_DATABASE_CONTROLLER_ENABLED\":\"true\", \
    \"REMOTE_CLUSTER_CONTROLLER_ENABLED\":\"true\"}}"
```

3. Configure a Ingress or Route, follow the instructions [here](setting_ingress_or_route_readme.md)

#### Part 2: participating cluster info preparation setup

Note:
  * This part should be done always, even if the cluster is already configured to run the Redis Enterprise Operator managed active-active databases.

1. Collect the REC credentials secret from all of the participating clusters, replase `<REC name>` with the local REC name:
```
  kubectl get secret -o yaml <REC name>
```
For example,  collecting the secret of a Redis Enterprise Cluster named "rec1" that resides in "ns1" namespace:
```
  kubectl get secret rec1 -o yaml
```
Following is an example of a REC crdentials secret named "rec1" with namespace "ns1":
```
apiVersion: v1
data:
  password: PHNvbWUgcGFzc3dvcmQ+
  username: PHNvbWUgdXNlcj4
kind: Secret
metadata:
  creationTimestamp: "2022-11-06T07:05:47Z"
  labels:
    app: redis-enterprise
    redis.io/cluster: rec1
  name: rec1
  namespace: ns1
  ownerReferences:
  - apiVersion: app.redislabs.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: RedisEnterpriseCluster
    name: rec1
    uid: b054ee49-299b-45db-be96-b7dfb543d531
  resourceVersion: "1883466"
  uid: d0ba47d2-5376-4e65-85c3-343c8f3227f5
type: Opaque
```

2. For each of the secrets you collected, generate a new secret with the credentials data and name it with the following convention: `redis-enterprise-<REC name>-<REC namespace>`, replace the `<REC name>` and the `<REC namespace>` with the Redis Enterprise cluster name and the namespace where it resides in respectively.
The new secret generated from the previous example of Redis Enterprise cluster named "rec1" that resides in "ns1" namespace should look as follows:
```
apiVersion: v1
data:
  password: PHNvbWUgcGFzc3dvcmQ+
  username: PHNvbWUgdXNlcj4
kind: Secret
metadata:
  name: redis-enterprise-rec1-ns1
type: Opaque
```

3. Apply all of the generated secrets from the previous step, replace `<generated secret file>` with a file containing each new secret you generated in the previous step.
```
  kubectl apply -f <generated secret file>
```

### Create the Active-Active database using Redis Enterprise Active Active Database (REAADB) and Redis Enterprise Remote Cluster (RERC) custom resources

On one of the participating clusters do the following:

1. Generate the RERC custom resources for each of the participating clusters.  
Following are examples of two participating clusters' corresponding RERC custom resources:  
a. The example for a REC named "rec1" in the namespace "ns1":
```
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseRemoteCluster
metadata:
  name: rec1.ns1
spec:
  apiFqdnUrl: test-example-api-rec1-ns1.redis.com
  dbFqdnSuffix: -example-cluster-rec1-ns1.redis.com
  secretName: redis-enterprise-rec1-ns1
```
b. The example for a REC named "rec2" in the namespace "ns2":
```
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseRemoteCluster
metadata:
  name: rec2.ns2
spec:
  apiFqdnUrl: test-example-api-rec2-ns2.redis.com
  dbFqdnSuffix: -example-cluster-rec2.ns2.redis.com
  secretName: redis-enterprise-rec2-ns2
```

Note:
  * View the Redis Enterprise Remote Cluster custom resource definition or the API doc for more details regarding the RERC fields.

2. Create the RERC custom resources you generated in the previous step and follow their statuses to verify the 'SPEC STATUS' is 'valid' and the 'STATUS' is 'active'.
To Create run the following:
```
  kubectl create -f <generated RERC file>
```
To follow the statuses of the applied RERC custom resources, run the following:
```
  kubectl get rerc <Created RERC name>
```
For example to view the status of the RERC named "rec1.ns1":
```
  kubectl get rerc rec1.ns1
```
The output should be as below:
```
  NAME        STATUS   SPEC STATUS   LOCAL
  rec1.ns1   Active   Valid         true
```

Note:
  * As the 'STATUS' and the 'SPEC STATUS' are 'Active' and 'Valid' respectively it means the configurations are correct. In case of an error, view the RERC custom resource events and the Redis Enterprise operator logs.

3. Generate the REAADB custom resource.
Following is an example of a REAADB custom resource named "example-aadb-1" that is linked to the REC named: "rec1" with two participating clusters.
```
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseActiveActiveDatabase
metadata:
  name: example-aadb-1
spec:
  redisEnterpriseCluster: 
    name: rec1
  participatingClusters:
    - name: rec1.ns1
    - name: rec2.ns2
```

Notes:
  * The REAADB name requirements are: Maximum of 63 characters, Only lower case letter, number, or hyphen (-) characters and Starts with a letter; ends with a letter or digit.
  * The "participatingClusters" contain the names of RERC custom resources that contain the cluster details we generated and created in the previous steps.

4. Create the REAADB custom resource you generated in the previous step and follow its statuses to verify the 'SPEC STATUS' is 'valid' and the 'STATUS' is 'active'.
To create, run the following:
```
  kubectl create -f <generated REAADB file>
```
To follow the statuses of the created REAADB custom resource run the following:
```
  kubectl get reaadb <created REAADB name>
```
For example, to view the status of the REAADB named "example-aadb-1":
```
  kubectl get reaadb example-aadb-1
```
The output should be as below:
```
  NAME             STATUS   SPEC STATUS   GLOBAL CONFIGURATIONS REDB   LINKED REDBS
  example-aadb-1   active   Valid                                      
```

Note:
  * As the 'STATUS' and the 'SPEC STATUS' are 'Active' and 'Valid' respectively it means the configurations are correct. In case of an error status, view the REAADB custom resource events and the Redis Enterprise operator logs.

5. View the REAADB and RERC custom resources that are automatically created on the participating clusters.
To get the REAADB run the following:
```
  kubectl get reaadb <REAADB name>
```
In addition, to view the RERCs for each RERC please run the following:
```
  kubectl get rerc <rerc name>
```

## Add participating cluster to an existing Active-Active database

**On the participating cluster you want to add in the namespace where the Redis Enterprise operator resides resides, please do the following:**  

1. Follow the instructions under: [Participating cluster preparation setup](#participating-cluster-preparation-setup).

2. Collect the REC credentials secret from the participating cluster you want to add, replace `<REC name>` with the local REC name):
```
  kubectl get secret -o yaml <REC name>
```
For example, collecting the secret of a Redis Enterprise cluster named "rec3" that resides in "ns3" namespace:
```
  kubectl get secret rec3 -o yaml
```
Following is an example of a REC credentials secret named "rec3" with namespace "ns3":
```
apiVersion: v1
data:
  password: PHNvbWUgcGFzc3dvcmQ+
  username: PHNvbWUgdXNlcj4
kind: Secret
metadata:
  creationTimestamp: "2022-11-06T07:05:47Z"
  labels:
    app: redis-enterprise
    redis.io/cluster: rec3
  name: rec3
  namespace: ns3
  ownerReferences:
  - apiVersion: app.redislabs.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: RedisEnterpriseCluster
    name: rec3
    uid: b054ee49-299b-45db-be96-b7dfb543d535
  resourceVersion: "1883465"
  uid: d0ba47d2-5376-4e65-85c3-343c8f3227f6
type: Opaque
```

3. For the secret you collected generate a new secret with the credentials data and name it with the following convention: `redis-enterprise-<REC name>-<REC namespace>`, replace the `<REC name>` and the `<REC namespace>` with the Redis Enterprise Cluster name and the namespace where it resides in respectively.
The new example credentials secret generated from the Redis Enterprise cluster named "rec3" that resides in "ns3" namespace would look like this:
```
apiVersion: v1
data:
  password: PHNvbWUgcGFzc3dvcmQ+
  username: PHNvbWUgdXNlcj4
kind: Secret
metadata:
  name: redis-enterprise-rec3-ns3
type: Opaque
```

4. On each of the participating clusters apply the generated secret from the previous step, replace `<generated secret file>` with a file containing the new secret you generated in the previous step.
```
  kubectl apply -f <generated secret file>
```

**On one of the participating clusters that already exists do the following:**  

1. Generate the RERC custom resource of the participating cluster that you want to add.
Following is an example of a RERC custom resource to add that contains the participating cluster details for REC named "rec3" in the namespace "ns3":
```
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseRemoteCluster
metadata:
  name: rec3.ns3
spec:
  apiFqdnUrl: test-example-api-rec3-ns3.redis.com
  dbFqdnSuffix: -example-cluster-rec3-ns3.redis.com
  secretName: redis-enterprise-rec3-ns3
```

Note:
  * View the Redis Enterprise Remote Cluster custom resource definition or the API doc for more details regarding the RERC fields.

2. Create the RERC custom resource you generated in the previous step and follow its statuses to verify the 'SPEC STATUS' is 'valid' and the 'STATUS' is 'active'.
To create run the following:
```
  kubectl create -f <generated RERC file>
```

To follow the statuses of the created rerc custom resources run the following:
```
  kubectl get rerc <Applied RERC name>
```
For example, to view the status of the RERC named "rec3.ns3":
```
  kubectl get rerc rec3.ns3
```
The output should be as below:
```
  NAME        STATUS   SPEC STATUS   LOCAL
  rec3.ns3   Active   Valid         true
```

Note:
  * As the 'STATUS' and the 'SPEC STATUS' are 'Active' and 'Valid' respectively it means the configurations are correct, in case of an error please view the RERC custom resource events and/ or the Redis Enterprise Operator logs.

3. Add and patch with the RERC name you created in the previous step on the 'participatingClusters' list in the REAADB spec. Follow the statuses to verify the 'SPEC STATUS' is 'valid' and the 'STATUS' is 'active'.
For example, adding to the REAADB named: "example-aadb-1" the RERC name: "rec3.ns3":
```
  kubectl patch reaadb example-aadb-1 --type merge --patch '{"spec": {"participatingClusters": [{"name": "rec3.ns3"}]}}'
```

4. View the REAADB 'participatingClusters' status and verify that the added participating cluster exists with it's corresponding ID.
To get the REAADB 'participatingClusters' status run the following:
```
  kubectl get reaadb <REAADB name> -o=jsonpath='{.status.participatingClusters}'
```
For example, to get the REAADB named: "example-aadb-1" participating clusters status:
```
  kubectl get reaadb example-aadb-1 -o=jsonpath='{.status.participatingClusters}'
```
Following is the output of the example above:
```
  [{"id":1,"name":"rec1.ns1"},{"id":2,"name":"rec2.ns2"},{"id":3,"name":"rec3.ns3"}]
```

## Remove participating cluster from an existing Active-Active database

**On one of the participating clusters that already exists do the following:**  

1. Remove the participating cluster from the REAADB "participatingClusters" spec.
You may use `kubectl edit` to remove the required participating cluster.
```
  kubectl edit reaadb <REAADB name>
```

**On one of the other participating clusters that exist do the following:**  

1. Follow the REAADB custom resource statuses and verify the 'SPEC STATUS' and 'status' are 'Valid' and 'active' respectively, and also, that the participating cluster removed from the 'participatingClusters' status.
To follow the statuses of the REAADB custom resource run the following:
```
  kubectl get reaadb <REAADB name> -o=jsonpath='{.status}'
```
For example, to view the status of the REAADB named "example-aadb-1" after remove the "rec3.ns3":
```
  kubectl get reaadb example-aadb-1 -o=jsonpath='{.status}'
```
the output should be as below:
```
  {... ,"participatingClusters":[{"id":1,"name":"rec1.ns1"},{"id":2,"name":"rec2.ns2"}],"redisEnterpriseCluster":"rec1","specStatus":"Valid","status":"active"}
```

**On the participating cluster that you removed do the following:**  

1. Verify that the REAADB custom resource was deleted and does not exist.
To get all the REAADB custom resources that exist, run the following:
```
  kubectl get reaadb -o=jsonpath='{range .items[*]}{.metadata.name}'
```

## Delete an existing Active-Active database

**On one of the participating clusters that already exists do the following:**  

1. Delete the REAADB of the active-active database.
To delete the REAADB run the following:
```
  kubectl delete reaadb <REAADB name>
```

**On all of the participating clusters do the following:**  

1. Verify that the REAADB custom resource was deleted and does not exist.
To get all the REAADB that exists run the following:
```
  kubectl get reaadb -o=jsonpath='{range .items[*]}{.metadata.name}'
```

## Update existing participating cluster (RERC) details

**On the participating cluster you want to modify (with the local RERC) do the following:**  

1. Patch the participating cluster corresponding local RERC that you want to update.
For example updating the DB FQDN suffix of an RERC named: "rec1.ns1":
```
  kubectl patch rerc rec1.ns1 --type merge --patch '{"spec": {"dbFqdnSuffix": "-example2-cluster-rec1-ns1.redis.com"}}'
```
To follow the statuses of the updated RERC custom resources run the following:
```
  kubectl get rerc <Updated RERC name>
```
For example to view the status of the RERC named "rec1.ns1":
```
  kubectl get rerc rec1.ns1
```
The output should be as below:
```
  NAME        STATUS   SPEC STATUS   LOCAL
  rec1.ns1   Active   Valid         true
```

Note:
  * As the 'STATUS' and the 'SPEC STATUS' are 'Active' and 'Valid' respectively it means the configurations are correct, in case of an error please view the RERC custom resource events and/ or the Redis Enterprise operator logs.

2. follow the REAADB custom resources statuses that are using this RERC to verify the 'SPEC STATUS' is 'valid' and the 'STATUS' is 'active'.
To view the statuses of the REAADB custom resources that is using this RERC run the following, replace `<REAADB name>` with the list of REAADB that is using this RERC seperated via spaces:
```
  kubectl get reaadb <REAADB names>
```
For example, to view the status of the REAADB named "example-aadb-1" and "example-aadb-2":
```
  kubectl get reaadb example-aadb-1 example-aadb-2
```
The output should be as below:
```
  NAME             STATUS   SPEC STATUS   GLOBAL CONFIGURATIONS REDB   LINKED REDBS
  example-aadb-1   active   Valid                                      
  example-aadb-2   active   Valid                                      
```

Note:
  * As the 'STATUS' and the 'SPEC STATUS' are 'Active' and 'Valid' respectively it means the configurations are correct. In case of an error status, view the REAADB custom resource events and the Redis Enterprise operator logs.

## Test your Active-Active database

The easiest way to test your Active-Active database is to set a key-value pair in one database and retrieve it from the other.

1. Retrieve the Active-Active database port from it's corresponding service.  
Get the corresponding Active-Active database service, replace  `<service name>` with the corresponding REAADB name:
```
  kubectl get svc <service name>
```
From the output fetch the redis 'targetPort':
```
  ports:
  - name: redis
    port: 16897
    protocol: TCP
    targetPort: 16897
```

2. From a pod within your cluster, use `redis-cli` to connect to your Active-Active database, replace `<service name>` with the corresponding REAADB name and `<port>` with  the port fetched from the previous step.
```
  redis-cli -h <service_name> -p <port>
```
 
 3. Set a test key with SET foo bar in the first database. If your Active-Active deployment is working properly, when connected to your second database, GET foo should output bar.
 
 to test externally you may use the instructions under 'Test your external access' [here](https://docs.redis.com/latest/kubernetes/re-databases/set-up-ingress-controller/)

## Limitations

### Support for only one Active-Active database with up-to three participating clusters across a set of participating clusters
Currently, there is only support for one managed Active-Active database, REAADB custom resource, with up-to three participating clusters across a set of participating clusters.

### No data path level configurations
Creating and updating the Active-Active database data level configurations (i.e. updating the DB size) is currently not supported via custom resources.
Updating an already created Active-Active database may be done through the RS API directly.

### Updating participating cluster secret
Updating the secret of a participating cluster is currently not supported via the operator, you may update it through the RS API directly and then modify the RERC secrets on all of the participating clusters.

### Unique participating clusters REC name.namespace
The participating clusters `<rec name.REC namespace>` must be different accross all of them.

### No REC upgrade
Upgrading Redis Enterprise cluster with operator managed Active-Active database is currently not supported.

### HashiCorp Vault secret storage
Storing the secrets in HashiCorp Vault is currently not supported for operator managed Active-Active database .