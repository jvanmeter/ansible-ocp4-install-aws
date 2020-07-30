# ansible-ocp4-install-aws
ansible-ocp4-install-aws is an ansible OpenShift 4 installer helper using the full stack AWS installation.  The playbooks allow for installation customization for cluster size, instance types, regions, along with other deployment options.

The ansible-ocp4-install-aws playbooks were originally developed to allow for simple and rapid OpenShift 4 cluster deployments for use with OpenShift training workshop lab environments.  This is especially true for the Service Mesh deployment, as the playbooks are designed to implement a specific workshop configuration.

---
## Dependencies

### OpenShift Pull Secret
Before starting the cluster install, you must obtain your OpenShift pull-secret file from cloud.redhat.com.  A Red Hat user account is required to log in.  The link below takes you directly to the pull-secret download page:

[OpenShift AWS Full Stack Installer](https://cloud.redhat.com/openshift/install/aws/installer-provisioned)

### Privileged AWS account
A privileged AWS account will be needed to install OpenShift 4 using the full stack installer.  The full stack installer attempts to simplify the OpenShift deployment process by fully orchestrating the creation and setup of all resources for the target environment.

The full stack deployer requires privileged permissions in AWS as it needs the ability to create and delete AWS users along with reading and writing IAM permissions.  This capability is often reserved for administrative users.  The specific permissions needed by the installer to deploy OpenShift 4.4 are provided in JSON format in the serviceAccount folder.  This JSON file can be imported into AWS to apply the necessary permissions to a user or service account.  Refer to the official OpenShift AWS installation documentation for additional information. [Configuring AWS User Permissions for OpenShift installation](https://docs.openshift.com/container-platform/latest/installing/installing_aws/installing-aws-account.html)

## Ansible playbook variables setup
All variables are contained in a single global variables file `./group_vars/all/all.yml`.  Many of the variables in the file can be left with their default settings.  The variables listed below should be reviewed and optionally changed based on the cluster requirements.

#### `openshift_version:`
It is recommended to use the latest stable openshift-installer build on the [OpenShift clients mirror site](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/).  The `openshift_version` variable should be the full client version such as for example `4.5.3`.  The installer has been tested with Release Candidate version strings (i.e. `4.5.0-rc.7`).

#### `openshift_odo_version:`
This is similar to the the `openshift_version` variable except it defines the ODO (OpenShift Do) CLI client version.  This is not required to deploy the workshop, however it is included as an additional tool.  Available ODO versions are located in [this directory](https://mirror.openshift.com/pub/openshift-v4/clients/odo/) on the OpenShift mirror site.

#### `openshift_cluster_admin:` and `openshift_user_password:`
Change these passwords to meet your desired security requirements.  All users will receive the same password.  As this is intended for ephemeral test and workshop environments, individual user passwords is not necessary.

#### `openshift_user_count:`
Specifies the number of users to provision.  All users interact with the same shared OpenShift cluster.  The default cluster configuration can host around 40 users without issue in a simple workshop environment.  Note that user accounts are enumerated user1 through userN where N is equal to the `openshift_user_count` value specified.

#### `openshift_cluster_name:` and `openshift_cluster_base_domain:`
The `openshift_cluster_name` is the name of the cluster and combined with the `openshift_cluster_base_domain` to build the FQDN of the cluster.  The domain name specified in `openshift_cluster_base_domain` is implicitly expected to be a domain you own and is managed in AWS Route 53.

#### `openshift_control_node_instance_type:` and `openshift_control_node_replicas:`
`openshift_control_node_replicas` refers to the number of OpenShift control nodes to deploy.  Currently, `3` control nodes per cluster is the only supported configuration.  Values greater or less than `3` are should not be used.

The default `openshift_control_node_instance_type` of `m5a.xlarge` does not need to be modified unless the number of workers is greater than `25`.  Refer to the [OpenShift Master node sizing](https://docs.openshift.com/container-platform/latest/scalability_and_performance/recommended-host-practices.html#master-node-sizing_) documentation for more information.

#### `openshift_worker_node_instance_type:` and `openshift_worker_node_replicas:`
The default `openshift_worker_node_instance_type` of `m5a.xlarge` does not need to be modified unless applications require more resources or specific instance capabilities (i.e. GPU acceleration).

`openshift_control_node_replicas` refers to the number of OpenShift worker nodes to deploy.  Generally, the default value of `3` worker nodes is sufficient for most workshops or smaller OpenShift proof of concept clusters.  Valid `openshift_control_node_replicas` can extend from `3` to `250` for typical clusters; although workshops rarely require more than `5` nodes of the `m5a.xlarge` size.  [OpenShift 4.x supports](https://docs.openshift.com/container-platform/latest/scalability_and_performance/planning-your-environment-according-to-object-maximums.html#cluster-maximums-major-releases_object-limits) up to `2,000` worker nodes.

#### `elasticsearch_operator_version:` and `keycloak_operator_version:`
Three of the five Service Mesh operators automatically select the correct version by using the `stable` operator version channel. The two exceptions are the Elasticsearch and Keycloak operators.

The `elasticsearch_operator_version` should match the `major.minor` release of the OpenShift version being deployed (i.e. `4.5` for OpenShift `4.5.3`).  Regular expression logic should make this selection automatically.

Specifying the current version for the `keycloak_operator_version` variable is a manual process.  The latest version can be found on the [Keycloak Operator](https://operatorhub.io/operator/keycloak-operator) page on OperatorHub.io.

## Ansible playbook overview
The ansible-ocp4-install-aws installation wrapper intent is to follow the steps outlined in the OpenShift install documentation, while automating most of the manual steps.  There are currently seven playbook files in this repo:
```
1_get_installation_files.yml
2_generate_ssh_keys.yml
3_install_ocp_cluster.yml
4_openshift_create_users.yml
5_service_mesh_install.yml
6_ocp_sm_workshop_setup.yml
7_openshift_teardown_cluster.yml
```
These playbooks are expected to be executed in sequential order.  Playbooks 1 through 4 are required for standing up a basic OpenShift cluster.  Playbooks 5 and 6 are related to deploying the OpenShift Service Mesh on the cluster.  Playbook 7 will completely teardown any assets associated with the cluster.  If the cluster was scaled out with additional nodes, these EC2 instances will be removed.

#### `sudo ansible-playbook 1_get_installation_files.yml`
Downloads and installs the following components:
- openshift-installer
- oc CLI platform admin tool
- odo CLI developer tool

#### `ansible-playbook 2_generate_ssh_keys.yml`
Generates ssh keys used to connect to the OpenShift cluster RHEL CoreOS nodes.  RHEL CoreOS is designed to be immutable and remotely managed.  Directly logging into the CoreOS nodes is generally discouraged.

#### `ansible-playbook 3_install_ocp_cluster.yml`
Creates the install-config.yaml file used by the openshift-installer to set the configuration variables used during cluster deployment.  These variables are located in the global variables file `./group_vars/all/all.yml`.

#### `ansible-playbook 4_openshift_create_users.yml`
Configures and enables the OpenShift HTPasswd authentication provider.  A cluster administrator and multiple workshop user accounts are generated and pushed to the OpenShift cluster.  The default kubeadmin account is disabled and removed once the new cluster administrator account is established.

#### `ansible-playbook 5_service_mesh_install.yml`
Installs the OpenShift Service Mesh operators to establish the service mesh services.  The operators installed to bring up the OpenShift Service Mesh are:
- Elasticsearch
- Istio
- Jaegar
- Kiali
- OpenShift Service Mesh

#### `ansible-playbook 6_ocp_sm_workshop_setup.yml`
Builds out the required configuration and setup for an OpenShift Service Mesh workshop.  If you are not hosting that specific workshop, you may want to consider reviewing this playbook instead of running it.

#### `ansible-playbook 7_openshift_teardown_cluster.yml`
Uses the openshift-installer to remove all OpenShift cluster assets provisioned during the initial cluster installation along with any cluster resources provisioned while it was running.

---
## Fedora 32 - Dependencies setup
Clone the ansible-ocp4-install-aws repo and change directory into the repo directory.

Place the pull secret file, named 'pull-secret' in the top level directory of the cloned repo

### Packages needed:
  - Ansible 2.9 or newer
  - AWS cli tool
  - Python3 libraries: 
    - python3-kubernetes
    - python3-openshift


The DNF package manager command to install the required components is provided below:

`sudo dnf ansible awscli python3-kubernetes python3-openshift`


