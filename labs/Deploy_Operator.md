# Deploy an Etcd Operator to OpenShift

The etcd operator manages etcd clusters deployed to Kubernetes and automates tasks related to establishing and operating an etcd cluster.

 - Create and Destroy
 - Resize
 - Failover
 - Rolling upgrade
 - Backup and Restore

Read [Best Practices](https://github.com/coreos/etcd-operator/blob/master/doc/best_practices.md) for more information on how to better use etcd operator.
This exercise requires you to have the `oc` cli which can be downloaded from `https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/`. Choose openshift-client tar that is suitable for your platform. The exercises are performed using this CLI and using a terminal (command-line) session.

## Step 1: Creating the Custom Resource Definition (CRD)

Perform the following steps:

1. Let's begin by logging in to OpenShift and creating a new project called operator-userXX (replace userXX with your lab userId)
Open a command line terminal window, create a directory for example `/tmp/operatorlab`, navigate into the folder

	```
	mkdir /tmp/operatorlab
	cd /tmp/operatorlab
	oc login https://api.ocp4faststart-aws.faststart.tk:6443/ --insecure-skip-tls-verify=true -u userXX -p <password>
	oc new-project operator-userXX
	```

2. Create the Custom Resource Definition (CRD) definition file the Etcd operator will be created to manage this Custom Resource:

	```
	cat > etcd-operator-crd.yaml<<EOF
	apiVersion: apiextensions.k8s.io/v1beta1
	kind: CustomResourceDefinition
	metadata:
	    name: etcdclusters.etcd.database.coreos.com
	spec:
	  group: etcd.database.coreos.com
	  names:
	    kind: EtcdCluster
	    listKind: EtcdClusterList
	    plural: etcdclusters
	    shortNames:
	    - etcdclus
	    - etcd
	    singular: etcdcluster
	  scope: Namespaced
	  version: v1beta2
	  versions:
	  - name: v1beta2
	    served: true
	    storage: true
	EOF
	```

3. Create the CRD from the above file:

	```
	oc create -f etcd-operator-crd.yaml
	```
  
	**Note**: Ignore the error: `Error from server (AlreadyExists)` as the resource is a Cluster wide resource, other user may have created the resource.

4. Verify the CRD was successfully created.

	```
	oc get crd | grep etcd
	```

5. Answer the following questions:

	- Which one of these `oc get` commands are valid to retrieve the list of Etcd clusters?  

		`oc get etcd` <br>	
`oc get etcdc`<br>
`oc get etcdcluster`<br>
`oc get etcdclusters`<br>
`oc get deployment etcdcluster`

	- From the valid commands in the list above, will the command retrieve all Etcd clusters, even the ones that are not created in your namespace (project)? ________

## Step 2: Creating the Service Account, Role, and RoleBinding

Typically, an operator needs additional access rights and authorities to interact with the OpenShift cluster to perform its automation. While granting system-wide authority such as `ClusterAdmin` would allow the operator to perform its function, it would also create a security exposure. Therefore it is recommended to give the operator granular access rights that are just enough for it to perform its function and not more.


1. Build a YAML file for a dedicated Service Account responsible for running the Etcd Operator:

	```
	cat > etcd-operator-sa.yaml<<EOF
	apiVersion: v1
	kind: ServiceAccount
	metadata:
	    name: etcd-operator-sa
	EOF
	```

2. Define the Service Account to OpenShift:

	```
	oc create -f etcd-operator-sa.yaml
	```

3. Verify that the Service Account was successfully created:

	```
	oc get sa
	```

4. Create a YAML file for the Role that contains all the access rights and authorities that the `etcd-operator-sa` Service Account need for performing actions against the Kubernetes API:

	```
	cat > etcd-operator-role.yaml<<EOF
	apiVersion: rbac.authorization.k8s.io/v1
	kind: Role
	metadata:
	  name: etcd-operator-role
	rules:
	- apiGroups:
	  - etcd.database.coreos.com
	  resources:
	  - etcdclusters
	  - etcdbackups
	  - etcdrestores
	  verbs:
	  - '*'
	- apiGroups:
	  - ""
	  resources:
	  - pods
	  - services
	  - endpoints
	  - persistentvolumeclaims
	  - events
	  verbs:
	  - '*'
	- apiGroups:
	  - apps
	  resources:
	  - deployments
	  verbs:
	  - '*'
	- apiGroups:
	  - ""
	  resources:
	  - secrets
	  verbs:
	  - get
	EOF
	```

5. Create the Role in OpenShift using the command:

	```
	oc create -f etcd-operator-role.yaml
	```

6. Verify that the Role was successfully created:

	```
	oc get roles
	```

7. Define the YAML file for binding the `etcd-operator-role` Role to the `etcd-operator-sa` Service Account:

	```
	cat > etcd-operator-rolebinding.yaml<<EOF
	apiVersion: rbac.authorization.k8s.io/v1
	kind: RoleBinding
	metadata:
	    name: etcd-operator-rolebinding
	roleRef:
	    apiGroup: rbac.authorization.k8s.io
	    kind: Role
	    name: etcd-operator-role
	subjects:
	- kind: ServiceAccount
	  name: etcd-operator-sa
	EOF
	```

8. Apply the YAML file to OpenShift, execute:

	```
	oc create -f etcd-operator-rolebinding.yaml
	```

9. Verify the RoleBinding was successfully created:

	```
	oc get rolebindings
	```

## Step 3: Creating the Etcd Operator Deployment

The Etcd operator, like any other OpenShift operator, runs as a pod inside a deployment. This implies that when the operator fails, it will automatically be restarted by the Replication Controller. This also allows you to restart the operator pod anytime by deleting the currently running operator pod.


1. Create the YAML file for the deployment containing the Etcd Operator container image. Note that the operator is run using the Service Account you created before:

	```
	cat > etcd-operator-deployment.yaml<<EOF
	apiVersion: extensions/v1beta1
	kind: Deployment
	metadata:
	    labels:
	      name: etcdoperator
	    name: etcd-operator
	spec:
	    replicas: 1
	    selector:
	      matchLabels:
	        name: etcd-operator
	    template:
	      metadata:
	        labels:
	          name: etcd-operator
	      spec:
	        containers:
	        - command:
	          - etcd-operator
	          - --create-crd=false
	          env:
	          - name: MY_POD_NAMESPACE
	            valueFrom:
	              fieldRef:
	                apiVersion: v1
	                fieldPath: metadata.namespace
	          - name: MY_POD_NAME
	            valueFrom:
	              fieldRef:
	                apiVersion: v1
	                fieldPath: metadata.name
	          image: quay.io/coreos/etcd-operator@sha256:c0301e4686c3ed4206e370b42de5a3bd2229b9fb4906cf85f3f30650424abec2
	          imagePullPolicy: IfNotPresent
	          name: etcd-operator
	        serviceAccountName: etcd-operator-sa
	EOF
	```

2. Create the Deployment object containing the operator, run:

	```
	oc create -f etcd-operator-deployment.yaml
	```

3. Verify that the Etcd Operator Deployment was successfully created:

	```
	oc get deployment
	```

4. Verify that the Etcd Operator Deployment pods are running:

	```
	oc get pods
	```

5. Open a new terminal window to follow Etcd Operator logs in real-time:

	```
	export ETCD_OPERATOR_POD=$(oc get pods -l name=etcd-operator -o jsonpath='{.items[0].metadata.name}')
	oc logs $ETCD_OPERATOR_POD -f
	```

6. Observe the leader-election lease on the Etcd Operator Endpoint:

	```
	oc get endpoints etcd-operator -o yaml
	```

	the message should be something similar to the following:
	```
	time="2020-01-22T17:45:15Z" level=info msg="Event(v1.ObjectReference{Kind:\"Endpoints\", Namespace:\"operator-user35\", Name:\"etcd-operator\", UID:\"f2bd4b44-3d3e-11ea-9abc-0694a43208c4\", APIVersion:\"v1\", ResourceVersion:\"1204325\", FieldPath:\"\"}): type: 'Normal' reason: 'LeaderElection' etcd-operator-696d48c895-l9ghj became leader"
	```

## Step 4: Creating the EtcdCluster Custom Resource (CR)

Now you have a running operator. The operator is just a program running in OpenShift that performs management actions against a specific CustomResourceDefinition. With the operator active, you can now deploy a resource defined by the operator. The resource will immediately get controlled and managed by the operator program in the operator pod.


1. Create a YAML file that defines an Etcd cluster by referring to the new Custom Resource, EtcdCluster, that was defined in the Custom Resource Definition on Step 1:

	```
	cat > etcd-operator-cr.yaml<<EOF
	apiVersion: etcd.database.coreos.com/v1beta2
	kind: EtcdCluster
	metadata:
	    name: example-etcd-cluster
	spec:
	    size: 3
	    version: 3.1.10
	EOF
	```

2. Create the Etcd cluster by running:

	```
	oc create -f etcd-operator-cr.yaml
	```

3. Verify the cluster object was created:

	```
	oc get etcdclusters
	```

4. Watch the pods in the Etcd cluster get created - it will follow the status progression of the Etcd cluster pods:

	```
	oc get pods -l etcd_cluster=example-etcd-cluster -w
	```

5. Verify the cluster has been exposed via a ClusterIP service:

	```
	oc get services -l etcd_cluster=example-etcd-cluster
	```

## Step 5:  Interacting with a Live Etcd Cluster

1. Let's grab one of the ectd pods

	```
	oc get pods -l etcd_cluster=example-etcd-cluster
	```

2. Access any of the pod from above:

	```
	oc rsh example-etcd-cluster-bvl9zpl55v
	```

3. Within the rsh session, set the etcd version and endpoint variables (the endpoint is the service for etcd cluster):

	```
	export ETCDCTL_API=3
	export ETCDCTL_ENDPOINTS=example-etcd-cluster-client:2379
	```

4. Within the rsh session, attempt to write a key/value into the Etcd cluster and retrieve it:

	```
	etcdctl put operator sdk
	etcdctl get operator
	```

5. Exit out of the client pod rsh session:

	```
	exit
	```

## Step 6 Scaling and Modifying the Etcd Version

1. Monitor the status of your pods. Open a new terminal session and login to OpenShift. *Remember*: change all userXX in the command to your userId. Arrange the display so that you can see both terminal sessions at the same time.

	```
	oc login https://api.ocp4faststart-aws.faststart.tk:6443/ --insecure-skip-tls-verify=true -u userXX -p <password>
	oc project operator-userXX
	oc get pods -l etcd_cluster=example-etcd-cluster -w
	```

2. Let's change the size of the Etcd example-etcd-cluster Custom Resource. Getting back to your original terminal session.
The Etcd Operator will detect the Custom Resource `spec.size` change and modify the number of pods in the cluster:

	```
	oc patch etcdcluster example-etcd-cluster --type='json' -p '[{"op": "replace", "path": "/spec/size", "value":5}]'
	```

3. Watch the pods in the Etcd cluster get created in your monitor session.

4. Let's change the version of our example-etcd-cluster Custom Resource. The etcd-operator pod will detect the Custom Resource `spec.version` change and create a new cluster with the newly specified image:

	```
	oc patch etcdcluster example-etcd-cluster --type='json' -p '[{"op": "replace", "path": "/spec/version", "value":3.2.13}]'
	```

5. Watch the pods in the Etcd cluster being updated, each pod in turn becoming unavailable and returning back to a Running state.

## Step 7: Disaster Recovery in Action

1. While the monitoring terminal session is still running, perform the following on your original terminal session:

	```
	export EXAMPLE_ETCD_CLUSTER_POD=$(oc get pods -l app=etcd -o jsonpath='{.items[0].metadata.name}')
	oc delete pod $EXAMPLE_ETCD_CLUSTER_POD
	```

2. Watch a pod in the Etcd cluster get removed and a new one get recreated until it becomes active again:

	```
	oc get pods -l etcd_cluster=example-etcd-cluster -w
	```

3. When the pod is recreated, is it the cluster (Kubernetes) automation (ie ReplicaSet) or the Operator doing it? _____
  Why? ____________________________________

## Step 8: Clean Up (Be careful)!
<details>
  <summary>Be Careful!</summary>

1. Delete your Etcd cluster:

	```
	oc delete etcdcluster example-etcd-cluster
	```

2. Delete the Etcd Operator:

	```
	oc delete deployment etcd-operator
	```

3. Do not delete the CRD as it is a shared resource.

4. Delete the Role, RoleBinding and ServiceAccount:

	```
	oc delete rolebinding/etcd-operator-rolebinding
	oc delete role/etcd-operator-role
	oc delete sa/etcd-operator-sa
	```

4. Delete the project - remember to change the userXX to your userId:

	```
	oc delete project operator-userXX
	```
</details>
