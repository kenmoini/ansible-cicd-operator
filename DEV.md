# Development Steps

> Note: Ansible Operator development is only supported on Linux

- Install the Operator SDK and other prerequisites: https://sdk.operatorframework.io/docs/building-operators/ansible/installation/
- Create a directory, `ansible-cicd-operator`
- Initialize the operator: `operator-sdk init --plugins=ansible --domain kemo.dev`
- Initialize the Git repo: `git init`
- Create an API for the core system requirements: `operator-sdk create api --group cicd --version v1alpha1 --kind CICDSystem --generate-role`
- Create an API for the application deployments: `operator-sdk create api --group cicd --version v1alpha1 --kind AppDeployment --generate-role`
- Create an API for the image registry definitions: `operator-sdk create api --group cicd --version v1alpha1 --kind ImageRegistry --generate-role`
- Create the Spec for the core system in `config/samples/cicd_v1alpha1_cicdsystem.yaml`
- Create the Spec for the image registry in `config/samples/cicd_v1alpha1_imageregistry.yaml`
- Create the Spec for the application deployment in `config/samples/cicd_v1alpha1_appdeployment.yaml`
- Set the default variables for the core system in `roles/cicdsystem/defaults/main.yml`
- Create the tasks for the core system in `roles/cicdsystem/tasks/main.yml`
- Test the operator so far by running it outside the cluster with `make install run`