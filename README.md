# CI/CD Operator

This is an Operator built with Ansible as the backend framework, meant to be a demonstration of how you can integrate different systems for easy application CI/CD.

It leverages a series of technologies such as OpenShift GitOps (ArgoCD), OpenShift Pipelines (Tekton), and Red Hat Advanced Cluster Management.  The combination of these other operators allows you to easily build and deploy applications across a hybrid cloud Kubernetes environment.

As it is currently architected, it relies upon being deployed to a Red Hat OpenShift cluster.  It could be deployed to other Kubernetes platforms but would require adjustement of the operator automation.

> There are development steps in [DEV.md](DEV.md)

## Usage

### Installing the Operator

1. Log into your OpenShift cluster
2. Deploy the Operator CatalogSource, eg via the CLI: `oc apply -f deploy/catalogsource.yml`
3. Once the CatalogSource syncs, you should be able to see the Operator available in the OperatorHub.
4. Either install the Operator via the OperatorHub in the OpenShift Web UI, or via the CLI: `oc apply -f deploy/subscription.yml`
5. Once the Operator is installed, you can begin to deploy the operator systems.

### Creating the cicd-system Namespce

While you can deploy most objects in any namespace you choose, a good default would be `cicd-system`.  This allows for better RBAC of resources.

`oc new-project cicd-system`

### Creating a CICDSystem

The CICDSystem CR defines the operation of the integrated technologies.  The specification allows for the deployment and integration of the dependencies such as ArgoCD, Tekton, and RHACM - however if they are already deployed then no action is taken.

There is only one CICDSystem object allowed per-cluster.

The example CICDSystem CR in the `config/samples/cicd_v1alpha1_cicdsystem.yaml` file is a good default.

`oc apply -n cicd-system -f config/samples/cicd_v1alpha1_cicdsystem.yaml`

### Creating GitRepo CRs

GitRepo CRs allow you to define Git repositories that can be pulled from and pushed to.  These objects are reusable and provide a secure way to connect to Git repositories.

Before creating a GitRepo CR, you must create a Secret for the object to reference.  This can be a Secret without any credentials, such as for public Git repos used for pulling assets, or you can use SSH Key based authentication for pulling/pushing to private repos.

```yaml
# Secret example with SSH key based authentication
---
kind: Secret
apiVersion: v1
metadata:
  name: git-credentials
  namespace: cicd-system
stringData:
  id_rsa:  |
    --------- PRIVATE KEY HERE ---------
  username: yourUsername
type: Opaque
```

With that Secret created you can create a GitRepo CR referencing a repo, default branch, and that credential:

```yaml
---
apiVersion: cicd.kemo.dev/v1alpha1
kind: GitRepo
metadata:
  labels:
    app.kubernetes.io/name: gitrepo
    app.kubernetes.io/instance: gitrepo-sample
    app.kubernetes.io/part-of: ansible-cicd-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: ansible-cicd-operator
  name: gitrepo-sample
  namespace: cicd-system
spec:
  url: git@github.com:kenmoini/cicd-system-apps.git
  branch: deploy
  credentials:
    name: git-credentials
    namespace: cicd-system
```

If this GitRepo is pointing to a repo that does not need authentication, obmit the `.spec.credentials` block.

Once the GitRepo CR is created, you can check its `.status.conditions` to verify if a connection has been successfully made.

### Creating ImageRegistry CRs

ImageRegistry CRs control where images are pushed to after they're built.  Similarly to GitRepo CRs, they need a Secret created beforehand - this is a typical `kubernetes.io/dockerconfigjson` type Secret:

```yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: image-registry-secret
  namespace: cicd-system
stringData:
  .dockerconfigjson: |
    {"auths":{"quay.io":{"auth":"eYyyy......==","email":""}}}
type: kubernetes.io/dockerconfigjson
```

Once the Secret has been made, you can create the ImageRegistry CR pointing to that Secret:

```yaml
---
apiVersion: cicd.kemo.dev/v1alpha1
kind: ImageRegistry
metadata:
  labels:
    app.kubernetes.io/name: imageregistry
    app.kubernetes.io/instance: imageregistry-sample
    app.kubernetes.io/part-of: ansible-cicd-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: ansible-cicd-operator
  name: imageregistry-sample
spec:
  credentials:
    name: image-registry-secret
    namespace: cicd-system
```

Once the CR is created, it will validate that the authentication secret is valid and usable which is reflected in the `.status.conditions`.

### Creating AppDeployment CRs

With the CICDSystem, an ImageRegistry, and a GitRepo or two, you can now continue to creating an AppDeployment CR.

AppDeployments currently support building from a Dockerfile/Containerfile, or by composing manifests in a repo via Kustomize.  Here are some examples of a 3 tier application with a frontend and backend being built via Dockerfile, and the MongoDB being deployed via Kustomize.

#### Frontend Application

```yaml
# make sure to modify the kustomize patch below to point to a valid FQDN in your cluster
---
apiVersion: cicd.kemo.dev/v1alpha1
kind: AppDeployment
metadata:
  labels:
    app.kubernetes.io/name: appdeployment
    app.kubernetes.io/instance: three-tier-frontend
    app.kubernetes.io/part-of: ansible-cicd-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: ansible-cicd-operator
  name: three-tier-frontend
spec:
  # sourceRepository controls the source code repository for the application
  sourceRepository:
    gitRepo:
      # The name/namespace of the GitRepo resource
      name: three-tier-source-repo
      namespace: cicd-system
    # contextDir is optional - if not defined will default to the root of the repository
    contextDir: frontend
    # branch can override the branch that is defined in the GitRepo CR
    branch: main
  # registry defines what image registry the built container will be pushed to
  registry:
    # The name/namespace of the ImageRegistry resource
    name: imageregistry-sample
    namespace: cicd-system
    # path is the path to the image in the registry after the registry URL defined in the ImageRegistry CR targeted Secret's Docker Config JSON
    path: kenmoini
    # imageName is optional - will default to the AppDeployment name
    imageName: acicdo-three-tier-frontend
  # build defines how the application is built
  build:
    # strategy: docker or kustomize
    strategy: docker
    # dockerfile: path to the dockerfile relative to the context_dir in the sourceRepository
    dockerfile: Dockerfile
    # deployment_type: Deployment or StatefulSet
    deployment_type: Deployment
    # kustomize allows you to pass extra parameters to the kustomization.yml file that is used to deploy this application
    kustomize:
      patches:
      - patch: |-
          - op: add
            path: "/spec/template/spec/containers/0/env/-"
            value:
              name: REACT_APP_BACKEND_URL
              value: 'https://three-tier-backend-three-tier-app.apps.mega-sno.v60.lab.kemo.network/api/tasks'
        target:
          kind: Deployment
          namespace: three-tier-app
          name: three-tier-frontend
  # create_networking will create a Service and Route for the application
  create_networking:
    # service will expose any port defined in the Dockerfile
    service: true
    # route will create a Route for the application
    route: true
    # hostname will set the hostname for the Route - if undefined will use the default wildcard ingress
    #hostname: my-awesome-app.com
  # target defines where the built application manifests are pushed to and what clusters they're deployed to
  target:
    cluster:
      # name and labelSelector are mutually exclusive - only one should be defined
      # name must be a cluster name found in ArgoCD/RHACM
      name: local-cluster
      # if a labelSelector is used an ApplicationSet will be generated instead of an Application
      #labelSelector:
      #  matchLabels:
      #    vendor: OpenShift
    # namespace is the namespace to deploy the application to on the target cluster
    namespace: three-tier-app
    # gitRepo is the name/namespace of the GitRepo resource to push manifests to
    gitRepo:
      # The name/namepsace of the GitRepo resource to push manifests to
      name: gitrepo-sample
      namespace: cicd-system
```

#### Backend Application

```yaml
---
apiVersion: cicd.kemo.dev/v1alpha1
kind: AppDeployment
metadata:
  labels:
    app.kubernetes.io/name: appdeployment
    app.kubernetes.io/instance: three-tier-backend
    app.kubernetes.io/part-of: ansible-cicd-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: ansible-cicd-operator
  name: three-tier-backend
spec:
  # sourceRepository controls the source code repository for the application
  sourceRepository:
    gitRepo:
      # The name/namespace of the GitRepo resource
      name: three-tier-source-repo
      namespace: cicd-system
    # contextDir is optional - if not defined will default to the root of the repository
    contextDir: backend
    # branch can override the branch that is defined in the GitRepo CR
    branch: main
  # registry defines what image registry the built container will be pushed to
  registry:
    # The name/namespace of the ImageRegistry resource
    name: imageregistry-sample
    namespace: cicd-system
    # path is the path to the image in the registry after the registry URL defined in the ImageRegistry CR targeted Secret's Docker Config JSON
    path: kenmoini
    # imageName is optional - will default to the AppDeployment name
    imageName: acicdo-three-tier-backend
  # build defines how the application is built
  build:
    # strategy: docker or kustomize
    strategy: docker
    # dockerfile: path to the dockerfile relative to the context_dir in the sourceRepository
    dockerfile: Dockerfile
    # deployment_type: Deployment or StatefulSet
    deployment_type: StatefulSet
    # kustomize allows you to pass extra parameters to the kustomization.yml file that is used to deploy this application
    kustomize:
      patches:
      - patch: |-
          - op: add
            path: "/spec/template/spec/containers/0/env/-"
            value:
              name: MONGO_CONN_STR
              value: mongodb://mongodb-svc:27017/todo?directConnection=true
          - op: add
            path: "/spec/template/spec/containers/0/env/-"
            value:
              name: MONGO_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-sec
                  key: username
          - op: add
            path: "/spec/template/spec/containers/0/env/-"
            value:
              name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-sec
                  key: password
        target:
          kind: StatefulSet
          namespace: three-tier-app
          name: three-tier-backend
  # create_networking will create a Service and Route for the application
  create_networking:
    # service will expose any port defined in the Dockerfile
    service: true
    # route will create a Route for the application
    route: true
    # hostname will set the hostname for the Route - if undefined will use the default wildcard ingress
    #hostname: my-awesome-app.com
  # target defines where the built application manifests are pushed to and what clusters they're deployed to
  target:
    cluster:
      # name and labelSelector are mutually exclusive - only one should be defined
      # name must be a cluster name found in ArgoCD/RHACM
      name: local-cluster
      # if a labelSelector is used an ApplicationSet will be generated instead of an Application
      #labelSelector:
      #  matchLabels:
      #    vendor: OpenShift
    # namespace is the namespace to deploy the application to on the target cluster
    namespace: three-tier-app
    # gitRepo is the name/namespace of the GitRepo resource to push manifests to
    gitRepo:
      # The name/namepsace of the GitRepo resource to push manifests to
      name: gitrepo-sample
      namespace: cicd-system
```

#### Database

```yaml
---
apiVersion: cicd.kemo.dev/v1alpha1
kind: AppDeployment
metadata:
  labels:
    app.kubernetes.io/name: appdeployment
    app.kubernetes.io/instance: three-tier-db
    app.kubernetes.io/part-of: ansible-cicd-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: ansible-cicd-operator
  name: three-tier-db
spec:
  # sourceRepository controls the source code repository for the application
  sourceRepository:
    gitRepo:
      # The name/namespace of the GitRepo resource
      name: three-tier-source-repo
      namespace: cicd-system
    # contextDir is optional - if not defined will default to the root of the repository
    contextDir: k8s_manifests/mongo
    # branch can override the branch that is defined in the GitRepo CR
    branch: main
  build:
    # strategy: docker or kustomize
    strategy: kustomize
    # with a kustomize build strategy the sourceRepository is used to pull the manifests and there are no futher configuration parameters
  # target defines where the built application manifests are pushed to and what clusters they're deployed to
  target:
    cluster:
      # name and labelSelector are mutually exclusive - only one should be defined
      # name must be a cluster name found in ArgoCD/RHACM
      #name: local-cluster
      # if a labelSelector is used an ApplicationSet will be generated instead of an Application
      labelSelector:
        matchLabels:
          vendor: OpenShift
    # namespace is the namespace to deploy the application to on the target cluster
    namespace: three-tier-app
    # gitRepo is the name/namespace of the GitRepo resource to push manifests to
    gitRepo:
      # The name/namepsace of the GitRepo resource to push manifests to
      name: gitrepo-sample
      namespace: cicd-system
```

Once those AppDeployment CRs are created, you'll find numerous Status Conditions added as it progresses through the build and deployment phases.