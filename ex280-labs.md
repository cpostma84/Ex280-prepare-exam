# EX280 Practice Labs — OpenShift Administrator

## Table of Contents

- [Lab 1 — HTPasswd Identity Provider](#lab-1--htpasswd-identity-provider)
- [Lab 2 — Managing Users](#lab-2--managing-users)
- [Lab 3 — Managing Groups](#lab-3--managing-groups)
- [Lab 4 — RBAC and Permissions](#lab-4--rbac-and-permissions)
- [Lab 5 — Custom Roles](#lab-5--custom-roles)
- [Lab 6 — Resource Quota](#lab-6--resource-quota)
- [Lab 7 — Limit Range](#lab-7--limit-range)
- [Lab 8 — Deploy from Resource Manifests](#lab-8--deploy-from-resource-manifests)
- [Lab 9 — Deploy from Images, Templates and Helm Charts](#lab-9--deploy-from-images-templates-and-helm-charts)
- [Lab 10 — Deploy Jobs for One-time Tasks](#lab-10--deploy-jobs-for-one-time-tasks)
- [Lab 11 — Manage Application Deployments](#lab-11--manage-application-deployments)
- [Lab 12 — ReplicaSets](#lab-12--replicasets)
- [Lab 13 — Labels and Selectors](#lab-13--labels-and-selectors)
- [Lab 14 — Configure Services](#lab-14--configure-services)
- [Lab 15 — Manage Secrets](#lab-15--manage-secrets)
- [Lab 16 — Manage Configuration Maps](#lab-16--manage-configuration-maps)
- [Lab 17 — Persistent Storage](#lab-17--persistent-storage)
- [Lab 18 — Storage Classes](#lab-18--storage-classes)
- [Lab 19 — StatefulSets](#lab-19--statefulsets)

---

## Lab 1 — HTPasswd Identity Provider

**Objective:** Configure HTPasswd as an identity provider and manage users

**Task details:**
1. Create an htpasswd file with users alice, bob, charlie and ted
2. Create a secret from the htpasswd file in the openshift-config namespace
3. Configure OAuth to use the htpasswd secret
4. Wait for the OAuth pods to restart
5. Verify the users and identities

### Solution

**Step 1 — Create the htpasswd file**
```bash
htpasswd -c -B -b htpasswd alice redhat
htpasswd -B -b htpasswd ted redhat
htpasswd -B -b htpasswd bob redhat
htpasswd -B -b htpasswd charlie redhat

# Verify
cat htpasswd
```

**Step 2 — Create the secret in openshift-config**
```bash
oc create secret generic my-htpass-secret \
  --from-file=htpasswd=htpasswd \
  -n openshift-config

# Verify
oc get secret my-htpass-secret -n openshift-config
```

**Step 3 — Configure OAuth**
```bash
oc get oauth cluster -o yaml > oauth.yaml
vi oauth.yaml
```

Ensure the spec section contains:
```yaml
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: my-htpass-secret
    mappingMethod: claim
    name: developer
    type: HTPasswd
```

```bash
oc replace -f oauth.yaml
```

**Step 4 — Wait for OAuth to restart**
```bash
oc get pods -n openshift-authentication -w
# Wait until: oauth-openshift-xxxx   1/1   Running
```

**Step 5 — Verify users and identities**
```bash
oc get users
oc get identities
```

---

## Lab 2 — Managing Users

**Objective:** Remove a user and change a user password

**Task details:**
1. Remove user ted from the htpasswd file and update the secret
2. Clean up the OpenShift user and identity objects for ted
3. Verify ted can no longer log in
4. Change the password of alice to 'password'
5. Verify alice can log in with the new password

### Solution

**Step 1 — Remove ted from htpasswd and update the secret**
```bash
htpasswd -D htpasswd ted

# Verify
cat htpasswd

# Update the secret
oc create secret generic my-htpass-secret \
  --from-file=htpasswd=htpasswd \
  --dry-run=client -o yaml \
  -n openshift-config | oc replace -f -
```

**Step 2 — Clean up OpenShift objects**
```bash
oc delete user ted
oc delete identity developer:ted
```

**Step 3 — Verify**
```bash
oc login -u ted -p redhat
# Should fail ❌

oc login -u alice -p redhat
# Should succeed ✅
```

**Step 4 — Change alice's password**
```bash
htpasswd -B -b htpasswd alice password

# Update the secret
oc create secret generic my-htpass-secret \
  --from-file=htpasswd=htpasswd \
  --dry-run=client -o yaml \
  -n openshift-config | oc replace -f -

# Wait for OAuth to reload
oc get pods -n openshift-authentication -w
```

**Step 5 — Verify new password**
```bash
oc login -u alice -p password
# Should succeed ✅
```

---

## Lab 3 — Managing Groups

**Objective:** Create groups, assign users and manage group permissions

**Task details:**
1. Create a new project called manage-groups
2. Create a group called testers with users alice, bob and charlie
3. Assign the view role to the testers group in the manage-groups project
4. Remove alice from the testers group
5. Verify only bob and charlie have access to the project

### Solution

**Step 1 — Create the project**
```bash
oc new-project manage-groups
```

**Step 2 — Create the group and add users**
```bash
oc adm groups new testers alice bob charlie
```

**Step 3 — Assign the view role to the group**
```bash
oc adm policy add-role-to-group view testers -n manage-groups
```

**Step 4 — Remove alice from the group**
```bash
oc adm groups remove-users testers alice
```

**Step 5 — Verify**
```bash
oc describe group testers
oc get rolebindings -n manage-groups
```

---

## Lab 4 — RBAC and Permissions

**Objective:** Manage cluster roles, project roles and self-provisioner permissions

**Task details:**
1. View existing cluster role bindings
2. Test whether a regular user can create projects by default
3. Remove the default self-provisioner rights from all OAuth users
4. Grant user alice explicit permission to create projects
5. Create a new project called permissions
6. Grant user bob admin rights in the permissions project
7. Create a group called developers and add user bob

### Solution

**Step 1 — View cluster role bindings**
```bash
oc get clusterrolebindings -o wide | grep self
oc describe clusterrolebinding.rbac self-provisioners
```

**Step 2 — Test as alice**
```bash
oc login -u alice -p password
oc new-project alice10
# Succeeds by default ✅
```

**Step 3 — Remove self-provisioner from all OAuth users**
```bash
oc login -u kubeadmin
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```

**Step 4 — Verify alice can no longer create projects**
```bash
oc login -u alice -p password
oc new-project alice10
# Should fail ❌
```

**Step 5 — Check who can create projects**
```bash
oc adm policy who-can create project
```

**Step 6 — Grant alice self-provisioner explicitly**
```bash
oc adm policy add-cluster-role-to-user self-provisioner alice

# Verify
oc login -u alice -p password
oc new-project alice22
# Should succeed ✅
```

**Step 7 — Create project and grant bob admin rights**
```bash
oc login -u kubeadmin
oc new-project permissions
oc policy add-role-to-user admin bob -n permissions

# Verify
oc get rolebinding -o wide -n permissions
```

**Step 8 — Create developers group and add bob**
```bash
oc adm groups new developers
oc adm groups add-users developers bob
```

---

## Lab 4b — User Roles and Permissions

**Objective:** Manage user roles and permissions in OpenShift

**Task details:**
- mariam: full cluster administration rights
- jane: able to create new projects
- john: not able to create new projects
- ian: view access to all projects in the cluster
- alice: not able to modify the cluster
- Ensure mariam is the only user with permission to modify the cluster

### Solution

```bash
# mariam: full cluster admin
oc adm policy add-cluster-role-to-user cluster-admin mariam

# jane: can create new projects
oc adm policy add-cluster-role-to-user self-provisioner jane

# john: cannot create new projects
oc adm policy remove-cluster-role-from-user self-provisioner john

# ian: view access to all projects
oc adm policy add-cluster-role-to-user view ian

# alice: cannot modify the cluster
oc adm policy remove-cluster-role-from-user self-provisioner alice
```

---

## Lab 4c — Project Permissions

**Objective:** Set up project permissions for users and groups

**Task details:**
- Create projects: avalon, atlantis, neptune, siren, triton, garuda, nagas
- jane: admin for avalon, atlantis and garuda
- john: view access for avalon and siren
- alice: edit access in avalon (excluding roles and rolebindings)
- Create group heroes (member: alice) with admin rights for neptune, siren, triton, nagas
- Create group wizards (member: ian) with view access for atlantis and edit access for siren

### Solution

**Create projects**
```bash
oc new-project avalon
oc new-project atlantis
oc new-project neptune
oc new-project siren
oc new-project triton
oc new-project garuda
oc new-project nagas
```

**Assign user permissions**
```bash
# jane as admin
oc adm policy add-role-to-user admin jane -n avalon
oc adm policy add-role-to-user admin jane -n atlantis
oc adm policy add-role-to-user admin jane -n garuda

# john view access
oc adm policy add-role-to-user view john -n avalon
oc adm policy add-role-to-user view john -n siren

# alice edit access
oc adm policy add-role-to-user edit alice -n avalon
```

**Create groups and assign permissions**
```bash
oc adm groups new heroes
oc adm groups new wizards

oc adm groups add-users heroes alice
oc adm groups add-users wizards ian

oc adm policy add-role-to-group admin heroes -n neptune
oc adm policy add-role-to-group admin heroes -n siren
oc adm policy add-role-to-group admin heroes -n triton
oc adm policy add-role-to-group admin heroes -n nagas

oc adm policy add-role-to-group view wizards -n atlantis
oc adm policy add-role-to-group edit wizards -n siren
```

**Verify**
```bash
oc get groups
oc get rolebindings -n avalon
oc get rolebindings -n siren
oc adm policy who-can get pods -n avalon | grep john
oc adm policy who-can create deployments -n avalon | grep alice
```

---

## Lab 5 — Custom Roles

**Objective:** Create a custom role and assign it to a user

**Task details:**
- Create a custom role viewnagas in the nagas project
- The role should provide read-only access to pods, deployments and imagestreams
- The role should permit get, list and watch
- Assign the viewnagas role to john

### Solution

```bash
# Create the custom role
oc create role viewnagas \
  --verb=get,list,watch \
  --resource=pods,deployments,imagestreams \
  -n nagas

# Assign the role to john
oc adm policy add-role-to-user viewnagas john -n nagas

# Verify
oc describe role viewnagas -n nagas
oc get rolebindings -n nagas
```

---

## Lab 6 — Resource Quota

**Objective:** Create a ResourceQuota for the neptune project

**Task details:**
- Create a ResourceQuota named neptune-quota in the neptune project
- Total memory consumption across all containers: 3Gi maximum
- Total CPU usage across all containers: 2 cores maximum
- Total memory requests across all containers: 1Gi maximum
- Maximum number of replication controllers: 3
- Maximum number of pods: 5
- Maximum number of services: 3

### Solution

```bash
# Create the ResourceQuota
oc create quota neptune-quota \
  --hard=limits.memory=3Gi,limits.cpu=2,requests.memory=1Gi,replicationcontrollers=3,pods=5,services=3 \
  -n neptune

# Verify
oc describe quota neptune-quota -n neptune
```

---

## Lab 7 — Limit Range

**Objective:** Create a LimitRange object in the avalon project

**Task details:**
- Create a LimitRange named avalon-constraints in the avalon project
- Pod memory limits: minimum 100Mi, maximum 500Mi
- Pod CPU limits: minimum 100m, maximum 500m
- Container memory limits: minimum 50Mi, maximum 300Mi
- Container CPU limits: minimum 50m, maximum 300m
- Default request: CPU 150m, memory 150Mi
- Verify the LimitRange works correctly

### Solution

**Step 1 — Switch to the avalon project**
```bash
oc project avalon
```

**Step 2 — Create the LimitRange**
```bash
cat << 'EOF' | oc apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: avalon-constraints
  namespace: avalon
spec:
  limits:
  - type: Pod
    max:
      memory: 500Mi
      cpu: 500m
    min:
      memory: 100Mi
      cpu: 100m
  - type: Container
    max:
      memory: 300Mi
      cpu: 300m
    min:
      memory: 50Mi
      cpu: 50m
    defaultRequest:
      memory: 150Mi
      cpu: 150m
EOF
```

**Step 3 — Verify**
```bash
oc describe limitrange avalon-constraints
```

**Step 4 — Verify defaults are injected automatically**
```bash
# Deploy a pod without resource limits
oc run nginx --image=bitnami/nginx --dry-run=client -o yaml > pod.yaml
oc apply -f pod.yaml

# Check that default requests were applied automatically
oc describe pod nginx | grep -A 5 Limits
```

---

## Lab 8 — Deploy from Resource Manifests

**Objective:** Deploy and manage an application using YAML manifests

**Task details:**
1. Generate a YAML manifest for a deployment named webserver using image bitnami/nginx in project manifests-demo
2. Edit the manifest to set replicas to 3
3. Apply the manifest to the cluster
4. Verify the pods are running
5. Update the manifest to change replicas to 5
6. Use dry-run to validate the change before applying
7. Check the diff between your local manifest and the cluster
8. Apply the update
9. Restart the deployment
10. Delete all resources using the manifest file

### Solution

```bash
# Step 1 — Create the project and generate the manifest
oc new-project manifests-demo

oc create deployment webserver --image=bitnami/nginx \
  -o yaml --dry-run=client --save-config > webserver.yaml

# Step 2 — Edit the manifest
vi webserver.yaml
# Change: replicas: 1 → replicas: 3

# Step 3 — Apply the manifest
oc apply -f webserver.yaml

# Step 4 — Verify
oc get pods -w

# Step 5 — Update replicas to 5
vi webserver.yaml
# Change: replicas: 3 → replicas: 5

# Step 6 — Validate with dry-run
oc apply -f webserver.yaml --dry-run=server --validate=true

# Step 7 — Check the diff
oc diff -f webserver.yaml

# Step 8 — Apply the update
oc apply -f webserver.yaml

# Step 9 — Restart the deployment
oc rollout restart deployment webserver

# Step 10 — Delete all resources
oc delete -f webserver.yaml
```

---

## Lab 9 — Deploy from Images, Templates and Helm Charts

**Objective:** Deploy applications using three different methods

**Task details:**

Part 1 — Deploy from image
1. Create a new project called deploy-demo
2. Deploy an application named myapp using the image docker.io/bitnami/nginx
3. Verify the pods are running

Part 2 — Deploy from OpenShift template
4. List the available templates in the openshift namespace
5. Examine the httpd-example template to see its parameters
6. Save the template to a local file
7. Process the template with parameter NAME=frontend and label app=web and apply it to the cluster
8. Verify the created resources

Part 3 — Deploy from Helm chart
9. Install Helm on your local machine
10. Add the bitnami Helm repo
11. Update the repo
12. Install WordPress with custom username and password
13. Verify the Helm release
14. Uninstall the Helm release

### Solution

**Part 1 — Deploy from image**
```bash
oc new-project deploy-demo
oc new-app --image=docker.io/bitnami/nginx --name=myapp --allow-missing-images
oc get pods -w
```

**Part 2 — Deploy from OpenShift template**
```bash
# List available templates
oc get templates -n openshift

# Examine the template
oc describe template httpd-example -n openshift

# Save the template to a local file
oc get template httpd-example -n openshift -o yaml > httpd-example-template.yaml

# Process and apply the template
oc process httpd-example -n openshift \
  -p NAME=frontend \
  -l app=web \
  -o yaml | oc apply -f -

# Verify
oc get all -l app=web
```

**Part 3 — Deploy from Helm chart**
```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add the bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install WordPress
helm install my-wordpress bitnami/wordpress \
  --set wordpressUsername=admin \
  --set wordpressPassword=redhat \
  -n deploy-demo

# Verify
helm list -n deploy-demo

# Uninstall
helm uninstall my-wordpress -n deploy-demo
```

---

## Lab 10 — Deploy Jobs for One-time Tasks

**Objective:** Create and manage a Job in OpenShift for a one-time task

**Task details:**
1. Generate a Job manifest named one-time-task using image bitnami/nginx that runs the command ls
2. Save the manifest to a file named myjob.yaml
3. Edit the manifest to add backoffLimit: 2 and ttlSecondsAfterFinished: 60
4. Deploy the job
5. Monitor the job status
6. View the logs of the job pod
7. Delete the job manually

### Solution

```bash
# Step 1+2 — Generate and save the manifest
oc create job one-time-task \
  --image=bitnami/nginx \
  --dry-run=client -o yaml \
  -- ls > myjob.yaml

# Step 3 — Edit the manifest
vi myjob.yaml
```

Ensure the spec section looks like this:
```yaml
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 60
  template:
    spec:
      containers:
      - name: one-time-task
        image: bitnami/nginx
        command: ["ls"]
      restartPolicy: Never
```

```bash
# Step 4 — Deploy the job
oc create -f myjob.yaml

# Step 5 — Monitor the job
oc get jobs
oc get pods

# Step 6 — View the logs
oc logs <pod-name>

# Step 7 — Delete the job
oc delete job one-time-task
```

---

## Lab 11 — Manage Application Deployments

**Objective:** Deploy, update, scale, rollback and monitor an application deployment

**Task details:**
1. Create a new project called deployment-demo
2. Create a deployment manifest named httpd-deployment.yaml using image nginxinc/nginx-unprivileged:stable with 2 replicas and rolling update strategy
3. Apply the manifest and verify the pods are running
4. Monitor the rollout status
5. Update the image to nginx-unprivileged:alpine
6. Monitor the rollout and verify the update
7. View the rollout history
8. Rollback to the previous version
9. Verify the rollback was successful
10. Scale the deployment to 5 replicas
11. Pause the deployment
12. Resume the deployment
13. Change the deployment strategy to Recreate and apply

### Solution

**Step 1 — Create the project**
```bash
oc new-project deployment-demo
```

**Step 2 — Create the manifest**
```bash
vi httpd-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: bitnamix/nginx-unprivileged:stable
```

```bash
# Step 3 — Apply and verify
oc apply -f httpd-deployment.yaml
oc get pods -w

# Step 4 — Monitor rollout
oc rollout status deployment httpd-deployment

# Step 5 — Update the image
oc set image deployment/httpd-deployment httpd=nginxinc/nginx-unprivileged:alpine

# Step 6 — Monitor and verify update
oc rollout status deployment httpd-deployment
oc get all

# Step 7 — View rollout history
oc rollout history deployment httpd-deployment
oc rollout history deployment httpd-deployment --revision=1
oc rollout history deployment httpd-deployment --revision=2

# Step 8 — Rollback to previous version
oc rollout undo deployment httpd-deployment

# Step 9 — Verify rollback
oc describe pod <pod-name> | grep Image

# Step 10 — Scale to 5 replicas
oc scale deployment httpd-deployment --replicas=5
oc get pods

# Step 11 — Pause the deployment
oc rollout pause deployment httpd-deployment

# Step 12 — Resume the deployment
oc rollout resume deployment httpd-deployment

# Step 13 — Change strategy to Recreate
vi httpd-deployment.yaml
# Change: type: RollingUpdate → type: Recreate
# Remove the rollingUpdate section entirely
oc apply -f httpd-deployment.yaml
```

---

## Lab 12 — ReplicaSets

**Objective:** Understand and manage ReplicaSets in OpenShift

**Task details:**
1. Create a ReplicaSet manifest named replicaset.yaml with name my-replicaset, 3 replicas, image docker.io/httpd:2.4 and label app=myapp
2. Apply the manifest and verify the pods are running
3. Check the status of the ReplicaSet
4. Scale the ReplicaSet to 5 replicas using oc scale
5. Scale back down to 2 replicas by editing the manifest
6. Delete one pod manually and verify the ReplicaSet recreates it
7. Create a Deployment named my-app-deployment with 3 replicas using image docker.io/bitnami/nginx
8. Observe the ReplicaSet created by the Deployment
9. Update the image to docker.io/nginx
10. Observe how the Deployment creates a new ReplicaSet and scales down the old one
11. Delete the ReplicaSet directly and observe what happens
12. Delete the Deployment

### Solution

**Step 1 — Create the ReplicaSet manifest**
```bash
vi replicaset.yaml
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: httpd
        image: docker.io/httpd:2.4
```

```bash
# Step 2 — Apply and verify
oc apply -f replicaset.yaml
oc get pods -w

# Step 3 — Check ReplicaSet status
oc get rs
oc describe rs my-replicaset

# Step 4 — Scale to 5 replicas
oc scale rs my-replicaset --replicas=5
oc get pods

# Step 5 — Scale down to 2 via manifest
vi replicaset.yaml
# Change: replicas: 3 → replicas: 2
oc apply -f replicaset.yaml
oc get pods

# Step 6 — Delete one pod manually
oc get pods
oc delete pod <pod-name>
oc get pods
# ReplicaSet recreates the pod automatically ✅

# Step 7 — Create a Deployment
oc create deployment my-app-deployment \
  --image=docker.io/bitnami/nginx \
  --replicas=3

# Step 8 — Observe the ReplicaSet
oc get rs
oc get all

# Step 9 — Update the image
oc set image deployment/my-app-deployment nginx=docker.io/nginx

# Step 10 — Observe new ReplicaSet
oc get rs
# New ReplicaSet scales up, old ReplicaSet scales to 0 ✅

# Step 11 — Delete the ReplicaSet directly
oc get rs
oc delete rs <replicaset-name>
oc get rs
# Deployment immediately recreates the ReplicaSet ✅

# Step 12 — Delete the Deployment
oc delete deployment my-app-deployment
oc delete rs my-replicaset
```

---

## Lab 13 — Labels and Selectors

**Objective:** Assign labels to resources, query with selectors and manage labels dynamically

**Task details:**
1. Create a Pod manifest named label-pod.yaml with labels app=my-app, environment=production and tier=frontend
2. Deploy the pod and verify the labels are assigned
3. Query pods by label app=my-app
4. Create a Deployment manifest named selector-deployment.yaml that uses a selector for label app=my-app
5. Update the environment label on the pod to staging
6. Verify the updated label
7. Remove the tier label from the pod
8. Verify the tier label is gone

### Solution

**Step 1 — Create the Pod manifest**
```bash
vi label-pod.yaml
```
oc run label-pod \
  --image=nginx \
  --labels="app=my-app,environment=production,tier=frontend" \
  --dry-run=client -o yaml > label-pod.yaml


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-labeled-pod
  labels:
    app: my-app
    environment: production
    tier: frontend
spec:
  containers:
  - name: httpd
    image: docker.io/httpd:2.4
```

```bash
# Step 2 — Deploy and verify
oc create -f label-pod.yaml
oc get pod my-labeled-pod --show-labels

# Step 3 — Query by label (equality-based)
oc get pod -l app=my-app

# Query by label (set-based)
oc get pod -l 'environment in (production,staging)'
```

**Step 4 — Create Deployment with selector**
```bash
vi selector-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: httpd
        image: docker.io/httpd:2.4
```

```bash
oc apply -f selector-deployment.yaml

# Step 5 — Update environment label
oc label pod my-labeled-pod environment=staging --overwrite=true

# Step 6 — Verify updated label
oc describe pod my-labeled-pod | grep Labels

# Step 7 — Remove tier label
oc label pod my-labeled-pod tier-

# Step 8 — Verify tier label is gone
oc describe pod my-labeled-pod | grep Labels
```

---

## Lab 14 — Configure Services

**Objective:** Deploy an application, expose it as a service and create a route for external access

**Task details:**

1. Create a new project called httpdemo
2. Deploy an application named my-http-app using image docker.io/bitnami/nginx
3. Verify the pods are running
4. Expose the application as a service
5. Verify the service is created
6. Create a route with hostname my-app.httpdemo.apps-crc.testing
7. Verify the route is created
8. Describe the route to see the details
9. Test the route by curling the hostname

### Solution

```bash
# Step 1 — Create the project
oc new-project httpdemo

# Step 2 — Deploy the application
oc create deployment my-http-app --image=docker.io/bitnami/nginx

# Step 3 — Verify pods
oc get pods -w

# Step 4 — Expose the application as a service
oc expose deployment my-http-app --port=8080

# Step 5 — Verify the service
oc get service

# Step 6 — Create a route
oc expose service my-http-app --hostname=my-app.httpdemo.apps-crc.testing

# Step 7 — Verify the route
oc get route

# Step 8 — Describe the route
oc describe route my-http-app

# Step 9 — Test the route
curl http://my-app.httpdemo.apps-crc.testing
```

---
## Lab 15 — Manage Secrets

**Objective:** Create, inspect and use OpenShift Secrets to securely manage sensitive application data

**Task details:**
1. Create a new project called `secrets-demo`
2. Create a generic Secret called `my-secret` with `username=myuser` and `password=mypassword`
3. Verify the Secret and inspect its keys
4. Create `username.txt` and `password.txt` and create a Secret called `my-file-secret` from these files
5. Create a Pod called `secret-pod` using the `busybox` image and inject the values from `my-secret` as environment variables
6. Verify the Secret values using the Pod logs
7. Create a Pod called `secret-volume-pod` and mount `my-secret` as files under `/etc/secrets`
8. Verify the Secret values from the mounted files
9. Create a Docker registry Secret called `registry-secret`
10. Create a TLS Secret called `my-tls-secret` using `tls.crt` and `tls.key`
11. Display `my-secret` as YAML and inspect the Base64 encoded values
12. Decode one of the Base64 encoded Secret values

### Solution

```bash
# Step 1 — Create the project
oc new-project secrets-demo

# Step 2 — Create a generic Secret
oc create secret generic my-secret \
  --from-literal=username=myuser \
  --from-literal=password=mypassword

# Step 3 — Verify the Secret
oc get secret my-secret
oc describe secret my-secret

# Step 4 — Create a Secret from files
echo -n 'myuser' > username.txt
echo -n 'mypassword' > password.txt

oc create secret generic my-file-secret \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Verify
oc describe secret my-file-secret

# Step 5 — Create a Pod using Secret environment variables
vi secret-pod.yaml
```

`secret-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: secret-container
    image: busybox
    command:
    - sh
    - -c
    - "echo Username: $USERNAME; echo Password: $PASSWORD"
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
```

```bash
# Create the Pod
oc apply -f secret-pod.yaml

# Step 6 — Verify the Secret values
oc get pod secret-pod
oc logs secret-pod

# Step 7 — Create a Pod with the Secret mounted as files
vi secret-volume-pod.yaml
```

`secret-volume-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: secret-container
    image: busybox
    command:
    - sh
    - -c
    - "cat /etc/secrets/username; echo; cat /etc/secrets/password"
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```

```bash
# Create the Pod
oc apply -f secret-volume-pod.yaml

# Step 8 — Verify the mounted files
oc logs secret-volume-pod

# Step 9 — Create a Docker registry Secret
oc create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword

# Verify
oc get secret registry-secret

# Step 10 — Create a TLS Secret
oc create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Verify
oc get secret my-tls-secret

# Step 11 — Inspect Secret data
oc get secret my-secret -o yaml

# Step 12 — Decode a Base64 encoded value
echo 'bXl1c2Vy' | base64 -d
# Output: myuser
```

### Key Concepts

```text
oc create secret generic
    → Generic Secret

--from-literal
    → Secret value directly from command line

--from-file
    → Secret value from file

secretKeyRef
    → Specific Secret key as environment variable

volumes.secret
    → Secret mounted as files

oc create secret docker-registry
    → Registry credentials

oc create secret tls
    → TLS certificate and key

Base64
    → Encoding, not encryption
```

---

## Lab 16 — Manage Configuration Maps

**Objective:** Create and use OpenShift ConfigMaps to manage non-sensitive application configuration

**Task details:**
1. Create a new project called `config-demo`
2. Create a ConfigMap called `my-config` with `APP_ENV=production` and `APP_DEBUG=false`
3. Verify the ConfigMap and inspect its data
4. Create a file called `index.html` containing a custom web page
5. Create a ConfigMap called `nginx-config` from the `index.html` file
6. Verify the ConfigMap contains the `index.html` key
7. Create an nginx Pod called `cm-nginx` using the `bitnami/nginx` image
8. Mount `nginx-config` as a volume at `/app`
9. Verify that `index.html` is available inside the Pod
10. Expose the Pod and use port forwarding to access the custom web page
11. Use `my-config` as environment variables in a Pod
12. Explain the difference between ConfigMaps and Secrets

### Solution

```bash
# Step 1 — Create the project
oc new-project config-demo

# Step 2 — Create a ConfigMap
oc create configmap my-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false

# Step 3 — Verify the ConfigMap
oc get configmap my-config
oc describe configmap my-config

# Step 4 — Create the custom web page
echo '<html><body><h1>Hello from OpenShift ConfigMap</h1></body></html>' > index.html

# Step 5 — Create a ConfigMap from the file
oc create configmap nginx-config \
  --from-file=index.html

# Step 6 — Verify the ConfigMap
oc get configmap nginx-config
oc describe configmap nginx-config

# Step 7-8 — Create the nginx Pod and mount the ConfigMap
vi cm-nginx-pod.yaml
```

`cm-nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-nginx
spec:
  containers:
  - name: nginx
    image: bitnami/nginx
    volumeMounts:
    - name: config-volume
      mountPath: /app
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

```bash
# Create the Pod
oc apply -f cm-nginx-pod.yaml

# Step 9 — Verify the mounted file
oc get pod cm-nginx
oc exec cm-nginx -- ls -l /app
oc exec cm-nginx -- cat /app/index.html

# Step 10 — Port forward to the nginx Pod
oc port-forward pod/cm-nginx 8080:8080
```

Open:

```text
http://localhost:8080
```

```bash
# Step 11 — Use the ConfigMap as environment variables
oc run cm-env-pod \
  --image=busybox \
  --restart=Never \
  --env-from=configmap/my-config \
  -- sh -c 'echo APP_ENV=$APP_ENV; echo APP_DEBUG=$APP_DEBUG'

# Verify
oc logs cm-env-pod

# Step 12 — Inspect the ConfigMap
oc get configmap my-config -o yaml
```

### Key Concepts

```text
oc create configmap
    → Create a ConfigMap

--from-literal
    → Configuration value directly from command line

--from-file
    → Configuration from a file

envFrom
    → Import ConfigMap keys as environment variables

configMap volume
    → Mount ConfigMap data as files

ConfigMap
    → Non-sensitive configuration

Secret
    → Sensitive data such as passwords and credentials

Base64
    → ConfigMaps store data as plain text
```

---

## Lab 17 — Persistent Storage

**Objective:** Create and use persistent storage with PersistentVolumes (PV), PersistentVolumeClaims (PVC) and Pods

**Task details:**
1. Create a new project called `storage-demo`
2. Create a directory `/mnt/data1` on the OpenShift node for HostPath storage
3. Create a PersistentVolume called `my-pv` with 1Gi capacity and ReadWriteOnce access
4. Create a PersistentVolumeClaim called `my-pvc` requesting 1Gi
5. Verify that the PVC is bound to the PV
6. Create a Pod called `persistent-pod` using the `bitnami/nginx` image
7. Mount `my-pvc` at `/app`
8. Verify that the Pod can access the persistent storage
9. Create a file in the mounted volume
10. Delete and recreate the Pod
11. Verify that the file still exists after the Pod is recreated

### Solution

```bash
# Step 1 — Create the project
oc new-project storage-demo

# Step 2 — Create the HostPath directory on the OpenShift node
oc debug node/crc -- chroot /host mkdir -p /mnt/data1

# Set the SELinux label
oc debug node/crc -- chroot /host chcon -Rt svirt_sandbox_file_t /mnt/data1

# Step 3 — Create the PersistentVolume
vi pv.yaml
```

`pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data1
```

```bash
# Create the PersistentVolume
oc apply -f pv.yaml

# Verify
oc get pv

# Step 4 — Create the PersistentVolumeClaim
vi pvc.yaml
```

`pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
# Create the PersistentVolumeClaim
oc apply -f pvc.yaml

# Step 5 — Verify the PVC is bound
oc get pvc
oc get pv

# Step 6-7 — Create the Pod and mount the PVC
vi persistent-pod.yaml
```

`persistent-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: persistent-pod
spec:
  containers:
  - name: nginx
    image: bitnami/nginx
    command:
    - sh
    - -c
    - "touch /app/persistent.txt && sleep 3600"
    volumeMounts:
    - name: data-volume
      mountPath: /app
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-pvc
```

```bash
# Create the Pod
oc apply -f persistent-pod.yaml

# Step 8 — Verify the mounted storage
oc get pod persistent-pod
oc exec persistent-pod -- df -h /app
oc exec persistent-pod -- ls -la /app

# Step 9 — Create a file in the persistent volume
oc exec persistent-pod -- sh -c 'echo "Persistent storage works" > /app/test.txt'
oc exec persistent-pod -- cat /app/test.txt

# Step 10 — Delete and recreate the Pod
oc delete pod persistent-pod
oc apply -f persistent-pod.yaml

# Step 11 — Verify the file still exists
oc exec persistent-pod -- cat /app/test.txt
```

Expected:

```text
Persistent storage works
```

### Key Concepts

```text
PersistentVolume (PV)
    → Storage resource provided by the cluster

PersistentVolumeClaim (PVC)
    → Request for storage made by an application

PV
    ↓
PVC
    ↓
Pod
    ↓
Mounted storage

ReadWriteOnce (RWO)
    → Volume can be mounted read/write by one node

HostPath
    → Storage backed by a directory on the node

Persistent storage
    → Data survives Pod deletion/recreation
```

---

## Lab 18 — Storage Classes

**Objective:** Use StorageClasses to dynamically provision persistent storage in OpenShift

**Task details:**
1. Create a new project called `storageclass-demo`
2. List the available StorageClasses
3. Identify the default StorageClass
4. Create a PersistentVolumeClaim called `my-hostpath-claim` using the default StorageClass
5. Request 1Gi of storage with `ReadWriteOnce` access
6. Verify that the PVC is created and becomes bound
7. Verify that OpenShift dynamically created a PersistentVolume
8. Create a Pod called `storage-pod` using the `bitnami/nginx` image
9. Mount the PVC at `/data`
10. Create a file in `/data` and verify that it is stored on the persistent volume
11. Inspect the StorageClass, PVC and PV
12. Explain the difference between a manually created PV and dynamically provisioned storage

### Solution

```bash
# Step 1 — Create the project
oc new-project storageclass-demo

# Step 2 — List the available StorageClasses
oc get storageclass

# Step 3 — Identify the default StorageClass
oc get storageclass

# Step 4-5 — Create a PVC using the default StorageClass
vi pvc.yaml
```

`pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-hostpath-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
# Create the PVC
oc apply -f pvc.yaml

# Step 6 — Verify the PVC
oc get pvc

# Step 7 — Verify the dynamically created PV
oc get pv

# Step 8-9 — Create a Pod using the PVC
vi storage-pod.yaml
```

`storage-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
  - name: nginx
    image: bitnami/nginx
    command:
    - sh
    - -c
    - "touch /data/storage-test.txt && sleep 3600"
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-hostpath-claim
```

```bash
# Create the Pod
oc apply -f storage-pod.yaml

# Verify the Pod
oc get pod storage-pod

# Step 10 — Create and verify a file
oc exec storage-pod -- sh -c 'echo "StorageClass works" > /data/storage-test.txt'
oc exec storage-pod -- cat /data/storage-test.txt

# Step 11 — Inspect the StorageClass, PVC and PV
oc get storageclass
oc describe pvc my-hostpath-claim
oc get pv
```

### Key Concepts

```text
StorageClass
    → Defines how storage is dynamically provisioned

PVC
    → Requests storage from a StorageClass

StorageClass
    ↓
PVC
    ↓
Dynamic PV
    ↓
Pod

Static provisioning
    → Administrator creates PV manually

Dynamic provisioning
    → PVC requests storage
    → StorageClass automatically provisions the PV
```

---

## Lab 19 — StatefulSets

**Objective:** Deploy and manage stateful applications using StatefulSets with stable identities and dedicated persistent storage

**Task details:**
1. Create a new project called `stateful-demo`
2. Create a headless Service called `mysql` with `clusterIP: None`
3. Create a StatefulSet called `mysql` with 2 replicas
4. Use the `bitnami/mysql` image
5. Configure the StatefulSet to use the `mysql` headless Service
6. Create a `volumeClaimTemplate` requesting 1Gi of storage for each Pod
7. Apply the Service and StatefulSet
8. Verify the StatefulSet, Pods and Service
9. Verify that each Pod has its own PersistentVolumeClaim
10. Scale the StatefulSet to 3 replicas
11. Verify that the new Pod receives its own PersistentVolumeClaim
12. Delete a Pod and verify that it is recreated with the same name

### Solution

```bash
# Step 1 — Create the project
oc new-project stateful-demo

# Step 2 — Create the headless Service
vi mysql-service.yaml
```

`mysql-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

```bash
# Create the Service
oc apply -f mysql-service.yaml

# Step 3-6 — Create the StatefulSet
vi mysql-statefulset.yaml
```

`mysql-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: bitnami/mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /bitnami/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

```bash
# Step 7 — Create the StatefulSet
oc apply -f mysql-statefulset.yaml

# Step 8 — Verify the StatefulSet, Pods and Service
oc get statefulset
oc get pods
oc get service

# Step 9 — Verify the PVCs and PVs
oc get pvc
oc get pv

# Step 10 — Scale the StatefulSet
oc scale statefulset mysql --replicas=3

# Step 11 — Verify the new Pod and PVC
oc get pods
oc get pvc

# Step 12 — Delete a Pod
oc delete pod mysql-0

# Verify that the Pod is recreated with the same name
oc get pods
```

### Key Concepts

```text
Deployment
    → Stateless applications
    → Pods have replaceable identities

StatefulSet
    → Stateful applications
    → Pods have stable identities

StatefulSet
    ↓
mysql-0
mysql-1
mysql-2

volumeClaimTemplates
    ↓
One PVC per Pod

Headless Service
    → clusterIP: None
    → Stable network identities

StatefulSet
    → Stable Pod names
    → Ordered creation/scaling
    → Dedicated storage per Pod
```

---
