# Development Steps

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

- Build and push the operator image: `make podman-build podman-push IMG=quay.io/kenmoini/cicd-operator:v0.0.1`
- Deploy the operator to the cluster `make deploy IMG=quay.io/kenmoini/cicd-operator:v0.0.1`
- Create CRDs and do testing
- Clean up with `make undeploy`

## Architecture

- The Ansible CI/CD Operator is meant to provide an easy way to standardize application development and deployment across Kubernetes environments.
- It leverages technologies such as OpenShift Pipelines (Tekton), Ansible Automation Platform 2, OpenShift Gitops (ArgoCD), and Red Hat Advanced Cluster Management to build and deploy applications across a hybrid cloud.
- Upon installing the Operator, you can configure a CICDSystem.  This Custom Resource will control the deployment and integration of the various systems.
- Once the CICDSystem is created, you can then configure a GitRepo CR.  This is where deployment manifests will be created after the applications are built and what is synced to various clusters by ArgoCD/RHACM.
- You will also need an ImageRegistry CR.  This will house the authentication details for an image registry where container images are stored.
- An AppDeployment CR will define what application is built, how it is built, which GitRepo and ImageRegistry CRs to leverage, and what clusters to deploy the application to.