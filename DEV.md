# Development Steps

## Architecture

- The Ansible CI/CD Operator is meant to provide an easy way to standardize application development and deployment across Kubernetes environments.
- It leverages technologies such as OpenShift Pipelines (Tekton), Ansible Automation Platform 2, OpenShift Gitops (ArgoCD), and Red Hat Advanced Cluster Management to build and deploy applications across a hybrid cloud.
- Upon installing the Operator, you can configure a CICDSystem.  This Custom Resource will control the deployment and integration of the various systems.
- Once the CICDSystem is created, you can then configure a GitRepo CR.  This is where deployment manifests will be created after the applications are built and what is synced to various clusters by ArgoCD/RHACM.
- You will also need an ImageRegistry CR.  This will house the authentication details for an image registry where container images are stored.
- An AppDeployment CR will define what application is built, how it is built, which GitRepo and ImageRegistry CRs to leverage, and what clusters to deploy the application to.

## Getting Started

> Note: Ansible Operator development is only supported on Linux

- Install the Operator SDK and other prerequisites: https://sdk.operatorframework.io/docs/building-operators/ansible/installation/
- Make sure the python modules are installed `python3 -m pip install ansible-runner-http`
- Create a directory, `ansible-cicd-operator`
- Initialize the operator: `operator-sdk init --plugins=ansible --domain kemo.dev`
- Initialize the Git repo: `git init`
- Install the needed binaries: `make ansible-operator` `make opm` `make kustomize`
- Replace the terms `docker` with `podman` in the Makefile

- Create an API for the core system requirements: `operator-sdk create api --group cicd --version v1alpha1 --kind CICDSystem --generate-role`
- Create an API for the application deployments: `operator-sdk create api --group cicd --version v1alpha1 --kind AppDeployment --generate-role`
- Create an API for the image registry definitions: `operator-sdk create api --group cicd --version v1alpha1 --kind ImageRegistry --generate-role`
- Create an API for the Git repo definitions: `operator-sdk create api --group cicd --version v1alpha1 --kind GitRepo --generate-role`

- Create the Spec for the core system in `config/samples/cicd_v1alpha1_cicdsystem.yaml`
- Create the Spec for the image registry in `config/samples/cicd_v1alpha1_imageregistry.yaml`
- Create the Spec for the Git repo in `config/samples/cicd_v1alpha1_gitrepo.yaml`
- Create the Spec for the application deployment in `config/samples/cicd_v1alpha1_appdeployment.yaml`

- Set the default variables for the core system in `roles/cicdsystem/defaults/main.yml`
- Create the tasks for the core system in `roles/cicdsystem/tasks/main.yml`
- Log into an OpenShift cluster with the CLI
- Test the operator so far by running it outside the cluster with `make install run`
- You may uninstall the operator after testing by running `make uninstall` - make sure to delete any created CICDSystem, ImageRegistry, GitRepo, and AppDeployment CRs first!

- Create a test namespace: `oc new-project cicd-system`
- Create the CICDSystem defined in `oc apply -f config/samples/cicd_v1alpha1_cicdsystem.yaml`

## In-cluster testing

Testing the operator in an active cluster is helpful to make sure all RBAC is in place and that your shell hasn't been passing environmental variables that wouldn't be present otherwise.

- Modify the `Makefile` to point to a valid image registry repository
- Build and push the operator image: `make podman-build podman-push`
- Deploy the operator to the cluster `make deploy`
- Create CRDs and do testing
- Clean up with `make undeploy`

## Creating the distributable Operator

- Ensure any instances have been deleted from testing and that the operator CRDs have been uninstalled `make uninstall` or `make undeploy` if you were running in-cluster tests
- Make sure to edit the `Makefile` to point to the proper repositories
- Build a fresh version of the operator: `make podman-build podman-push`
- Run `make bundle` and answer the prompts
- Build and push the bundle image: `make bundle-build bundle-push`
- Optionally add an image to the generated ClusterServiceVersion in the `config/manifests/base` directory under `.spec.icon`
- Validate the bundle: `make bundle-validate`
- Build the catalog: `make catalog-build catalog-push`
- Deploy the CatalogSource: `oc apply -f deploy/catalogsource.yml`
- Observe the Operator available in the OperatorHub
- Create the Subscription to create the Operator deployment `oc apply -f deploy/subscription.yml`

## Upgrading an Operator/Bundle/Catalog

- Bump the `VERSION` in the `Makefile`
- Run `make podman-build podman-push`
- Run `make bundle bundle-build bundle-push`
- Run `make catalog-build catalog-push`
- Update the version tag in the `deploy/catalogsource.yml` file
- Deploy the updated CatalogSource: `oc apply -f deploy/catalogsource.yml`