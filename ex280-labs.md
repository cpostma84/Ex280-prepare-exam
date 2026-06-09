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
oc new-app --image docker.io/bitnami/nginx --allow-missing-images
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
2. Create a deployment manifest named httpd-deployment.yaml using image docker.io/httpd:2.4 with 2 replicas and rolling update strategy
3. Apply the manifest and verify the pods are running
4. Monitor the rollout status
5. Update the image to docker.io/httpd:2.4-alpine
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
        image: docker.io/httpd:2.4
```

```bash
# Step 3 — Apply and verify
oc apply -f httpd-deployment.yaml
oc get pods -w

# Step 4 — Monitor rollout
oc rollout status deployment httpd-deployment

# Step 5 — Update the image
oc set image deployment/httpd-deployment httpd=docker.io/httpd:2.4-alpine

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
oc new-app --name=my-http-app --image=docker.io/bitnami/nginx

# Step 3 — Verify pods
oc get pods -w

# Step 4 — Expose as service
oc expose deployment my-http-app --port=8080

# Step 5 — Verify service
oc get service

# Step 6 — Create a route
oc expose service my-http-app \
  --hostname=my-app.httpdemo.apps-crc.testing

# Step 7 — Verify route
oc get route

# Step 8 — Describe the route
oc describe route my-http-app

# Step 9 — Test the route
curl http://my-app.httpdemo.apps-crc.testing
```
