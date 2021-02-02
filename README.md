# Deploy OPA (Open Policy Agent) on a Kubernetes cluster

In this project, we will deploy OPA on a GKE cluster and set some policies to allow user to deploy images only from a specified registry.

<br>


> Estimated time for the deployment : **20-30 min**

## Quick reminder
- `OPA (Open Policy Agent) Gatekeeper`: 
    - It's a policy engine that will help you enforce policies and restrict object creation/update/delete on your Kubernetes cluster
    - Gatekeeper is different from the previous version OPA with sidecar kube-mgmt (aka Gatekeeper v1)
<br>

- `How it works` :
   - Gatekeeper uses an OPA Constraint Framework 
    - `Constraint Templates`: Where you will define your constraint model using rego engine [more info](https://www.openpolicyagent.org/docs/latest/#rego)
    - `Constraint` : It's actually the implementation of your model
<br>

- `Good to know` : You don't have to write Constraint Templates and Constraint from scratch, you can use pre-existing policies [Gatekeeper library](https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/general)

## Pre-requisites
- Kubernetes cluster already running (you can follow this [tutorial](https://gitlab.com/treeptik/community-of-practice/cop-infra-cloud-container/deploy-gke) to create a GKE cluster)

<br>



**Deploy OPA (Gatekeeper) on your GKE cluster**
```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.2/deploy/gatekeeper.yaml

Output

namespace/gatekeeper-system created
customresourcedefinition.apiextensions.k8s.io/configs.config.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constraintpodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constrainttemplatepodstatuses.status.gatekeeper.sh created
customresourcedefinition.apiextensions.k8s.io/constrainttemplates.templates.gatekeeper.sh created
serviceaccount/gatekeeper-admin created
podsecuritypolicy.policy/gatekeeper-admin created
role.rbac.authorization.k8s.io/gatekeeper-manager-role created
clusterrole.rbac.authorization.k8s.io/gatekeeper-manager-role created
rolebinding.rbac.authorization.k8s.io/gatekeeper-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/gatekeeper-manager-rolebinding created
secret/gatekeeper-webhook-server-cert created
service/gatekeeper-webhook-service created
deployment.apps/gatekeeper-audit created
deployment.apps/gatekeeper-controller-manager created
validatingwebhookconfiguration.admissionregistration.k8s.io/gatekeeper-validating-webhook-configuration created
```

**Create a ConstraintTemplate**
`template.yaml`
```
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
  annotations:
    description: Requires container images to begin with a repo string from a specified
      list.
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }
```
```
kubectl apply -f template.yaml

Output

constrainttemplate.templates.gatekeeper.sh/k8sallowedrepos created
```



**Create a constraint**
`constraint.yaml`
```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: repo-is-gcrio
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "default"
  parameters:
    repos:
      - "gcr.io"
```
- `repos` : list of restricted repos (only images from this repo will be allowed to be deployed on your cluster), in our example grc.io
```
kubectl apply -f constraint.yaml

Output

k8sallowedrepos.constraints.gatekeeper.sh/repo-is-gcrio created
```


## Build a docker image and push it to the gcloud container registry grc.io

**Build the docker image with nginx (from Dockerfile)**

**Dockerfile**
```
cat > Dockerfile <<EOF
    FROM ubuntu:18.04
    RUN apt-get update
    RUN apt-get install nginx -y
    COPY index.html /var/www/html/
    EXPOSE 80
    CMD ["nginx","-g","daemon off;"]
EOF
```

`Make sure to have in the same directory your Dockerfile and the index.html before running the command`
```
docker build -t nginxdemo:1.0 .
```

**Validate that the image is created**
```
docker images

Output

REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
nginxdemo    1.0       d92b02a6776e   6 seconds ago   159MB
ubuntu       18.04     c090eaba6b94   7 days ago      63.3MB
```

**Tag and push your image to gcloud container registry gcr.io**
```
docker tag nginxdemo:1.0 gcr.io/gke-demo-cluster-dev/nginxdemo:1.0
docker push gcr.io/gke-demo-cluster-dev/nginxdemo:1.0
```
- `gke-demo-cluster-dev` : make sure to replace with your project id


## Validate that the policy works (meaning you can only deploy images from gcr.io/)

**Deploy a pod from the gcr.io registry** 
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-good-repo
spec:
  containers:
  - name: nginx-good-repo
    image: gcr.io/gke-demo-cluster-dev/nginxdemo:1.0
EOF

Output

pod/nginx-good-repo created
```

**Deploy a pod from another registry** 
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-bad-repo
spec:
  containers:
  - name: nginx-bad-repo
    image: nginx:1.16.1
EOF

Output

Error from server ([denied by repo-is-gcrio] container <nginx-bad-repo> has an invalid image repo <nginx:1.16.1>, allowed repos are ["gcr.io"]): error when creating "STDIN": admission webhook "validation.gatekeeper.sh" denied the request: [denied by repo-is-gcrio] container <nginx-bad-repo> has an invalid image repo <nginx:1.16.1>, allowed repos are ["gcr.io"]
```

**Resources used to complete this project**
- https://github.com/open-policy-agent/gatekeeper
- https://www.openpolicyagent.org/docs/latest/#rego
- https://github.com/open-policy-agent/gatekeeper-library