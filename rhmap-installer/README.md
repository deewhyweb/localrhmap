
# RHMAP Ansible

This repository contains [Ansible](https://www.ansible.com) Playbooks to deploy [RHMAP](https://www.redhat.com/en/technologies/mobile/application-platform) Core and [RHMAP](https://www.redhat.com/en/technologies/mobile/application-platform) MBaaS to your [OpenShift](https://www.openshift.com/container-platform/) cluster.


<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Getting Started](#getting-started)
  - [Prerequisite](#prerequisite)
  - [Configuring your inventory file](#configuring-your-inventory-file)
  - [Notes about logging in to the master node](#notes-about-logging-in-to-the-master-node)
- [Deploying RHMAP Core](#deploying-rhmap-core)
  - [Prerequisites](#prerequisites)
  - [Setting variables](#setting-variables)
    - [Configure Monitoring Components](#configure-monitoring-components)
    - [Configure Front End Components](#configure-front-end-components)
  - [Running the Playbook to deploy RHMAP Core](#running-the-playbook-to-deploy-rhmap-core)
  - [Tags](#tags)
- [Deploy RHMAP 1-Node MBaaS](#deploy-rhmap-1-node-mbaas)
  - [Prerequisites](#prerequisites-1)
  - [Setting variables](#setting-variables-1)
  - [Running the Playbook to deploy RHMAP 1-Node MBaaS](#running-the-playbook-to-deploy-rhmap-1-node-mbaas)
  - [Tags](#tags-1)
- [Deploy RHMAP 3-Node MBaaS](#deploy-rhmap-3-node-mbaas)
    - [Prerequisites](#prerequisites-2)
    - [Setting variables](#setting-variables-2)
    - [Running the Playbook to deploy RHMAP 3-Node MBaaS](#running-the-playbook-to-deploy-rhmap-3-node-mbaas)
    - [Tags](#tags-2)
- [Deploy RHMAP Self-Managed Buildfarm](#deploy-rhmap-self-managed-buildfarm)
  - [Prerequisites](#prerequisites-3)
  - [Setting variables](#setting-variables-3)
  - [Running the Playbook to deploy RHMAP Buildfarm](#running-the-playbook-to-deploy-rhmap-buildfarm)
  - [Tags](#tags-3)
- [Working With Proxies](#working-with-proxies)
- [Registering With Subscription Manager](#registering-with-subscription-manager)
- [Upgrading RHMAP](#upgrading-rhmap)
- [Seeding Docker Images](#seeding-docker-images)
  - [Dependencies](#dependencies)
- [POC Playbook](#poc-playbook)
  - [Prerequisites:](#prerequisites)
  - [Deploying The POC Projects](#deploying-the-poc-projects)
- [Running with Docker](#running-with-docker)
  - [Docker With OpenShift Cluster Up](#docker-with-openshift-cluster-up)
- [Troubleshooting and useful Information](#troubleshooting-and-useful-information)
  - [Using Tags](#using-tags)
  - [Modifying Variables](#modifying-variables)
  - [Options](#options)
  - [Configuring the log path](#configuring-the-log-path)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Getting Started

### Prerequisite

1. Ansible 2.2.0 or later: The code has been tested against 2.2.0 and 2.2.1 only. Earlier versions may break "include" statements in the Playbooks.

    **For installation instructions for your OS see [Ansible Installation Guide](http://docs.ansible.com/ansible/intro_installation.html)**


### Configuring your inventory file

Each users [inventory file](http://docs.ansible.com/ansible/intro_inventory.html) will vary depending on the type of install required and the information specific to the users [OpenShift](https://www.openshift.com/container-platform/) cluster.

Users must set values relative to their infrastructure and configuration. For simplicity, there are a number of default templates which you can use to get started, located at `inventories/examples/*`. Choose the template that best fits your use case to start with. The following variables should be set in your inventory.

| Variable           | Description                                                                                                                                |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| `ansible_ssh_user` | This variable sets the SSH user for the installer to use. This user should allow SSH-based authentication without requiring a password     |
| `ansible_sudo`     | Must be set to true. Allows to force privilege escalation when required                                                                    |
| `target`           | The type of installation required - i.e. `enterprise` or `dedicated`    															          |
| `cluster_hostname` | The OpenShift router subdomain. This value should be set to the value of `routingConfig.subdomain` from the OpenShift `master-config.yaml` |
| `domain_name`      | The custom subdomain to be used for RHMAP                                                                                                  |
| `url_to_check`     | An external URL for sanity checking to ensure the **required** outbound HTTP access is available. Default value is `https://www.npmjs.com `|
| `kubeconfig`       | Path to config file on the master node containing the  `system:admin` client certs. Default value is `/etc/origin/master/admin.kubeconfig` |
|`login_url`         | The URL of the Openshift master which the login will be carried out on. Needs to be explicitly set when running the playbook with `ansible_connection=local` is set. Should not be set if the master node is behind a proxy. |
|`master_url`        | The URL of the OpenShift master node - to be used in the poc playbook for configuration purposes.                                                                                                      |

When the variable `target` is set to `dedicated` an additional variable must be set in the inventory.

| Variable           | Description                                                                                                                                |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
|`ansible_connection`| This must be included and the value set to `local`                                                                                         |

When using [Gluster FS](https://www.gluster.org) for persistent storage, the folowing variables should be set in the inventory.

gluster_endpoints

| Variable              | Description                                                                                                                                |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
|`gluster_storage`      | This must be set to `true`. Default value is `false`                                                                                       |
|`gluster_metadata_name`| The name that defines the Gluster cluster in [Persistent Volume](https://tinyurl.com/k68vmfp) definition.                                  |
|`gluster_endpoints`    | A commaa seperated list of IP values. Must be the actual IP addresses of a Gluster server, not FQDN's.                                     |

Having configured all the above variables, you must add the hostname or IP Address of each Node in your OpenShift cluster under the appropriate group - (`master`, `core`, `mbaas`)

**Note** If your OpenShift cluster has multiple master nodes for high availability, it is only necessary to supply a single value under the `master` group.

### Notes about logging in to the master node

**Remote host** 

1. If you are running the playbook locally but pointing at a remote master node (i.e. when `ansible_connection=local`) then login_url must be set when running the playbook. This is the address the playbook will use to oc login to the master node.

2. If you are running the playbook remotely (i.e. while in an ssh session) and the master node is not behind a proxy, then either a login_url can be provided or the login role will default to localhost.

3. If you are running the playbook remotely (i.e. while in an ssh session) and the master node is behind a proxy, then **do not** set a login_url variable as localhost will be used to login.

***

## Deploying RHMAP Core

### Prerequisites

1. Nodes must be registered with [Red Hat Subscription Manager](https://access.redhat.com/documentation/en-US/Red_Hat_Subscription_Management/1/html-single/RHSM)

	**See [Registering With Subscription Manager](#registering-with-subscription-manager) to do this with the Ansible Playbook included in this repo**

2. Installation of RHMAP Core in production will require the following resources outlined in the table below at a minimum:


	| Description                                | Variable name                | Default value           | Fail/warning/recommended             |
	|--------------------------------------------|------------------------------|-------------------------|--------------------------------------|
	| Min number of CPUs                         | `min_required_vCPUS`         | 4                       | fail only in strict_mode             |
	| Min system memory per node (in MB)         | `required_mem_mb_threshold`  | 7000                    | fail only in strict_mode             |
	| Min total free memory of all nodes (in KB) | `warning_kb_value`           | 4000000                 | warning                              |
	| Number of PVs with 50 GB storage           | `required_50_pv`             | 0                       | warning                              |
	| Number of PVs with 25 GB storage           | `required_25_pv`             | 2                       | warning                              |
	| Number of PVs with  5 GB storage           | `required_5_pv`              | 2                       | warning                              |
	| Number of PVs with  1 GB storage           | `required_1_pv`              | 1                       | warning                              |

### Setting variables

The variables required for installation of RHMAP Core are set in `roles/deploy-core/defaults/main.yml`

#### Configure Monitoring Components

Setup the monitoring parameters with SMTP server details. This is required to enable email alerting via Nagios when a monitoring check fails. If you don’t require email alerting or want to set it up at a later time, the sample values can be used.

```yaml
monitoring:
  smtp_server: "localhost"
  smtp_username: "username"
  smtp_password: "password"
  smtp_from_address: "nagios@example.com"
  rhmap_admin_email: "root@localhost"
```

#### Configure Front End Components

* SMTP server parameters

The platform sends emails for user account activation, password recovery, form submissions, and other events. Set the following variables as appropriate for your environment.

```yaml
frontend:
  smtp_server: "localhost"
  smtp_username: "username"
  smtp_password: "password"
  smtp_port: "25"
  smtp_auth: "false"
  smtp_tls: "false"
  email_replyto: "noreply@localhost"
 ```

* Git External Protocol

This is set to `https` as default which is typical for production use cases. If you wish to use a self signed certificate however. This should be changed to `http`

 ```yaml
frontend:
  git_external_protocol: "https"
  builder_android_service_host: "https://androidbuild.feedhenry.com"
  builder_iphone_service_host: "https://iosbuild.feedhenry.com"
```

* SaaS Build Farm Configuration

To determine the values for the `builder_android_serviced_host` and `builder_iphone_serviced_host` variables, contact [Red Hat Support](https://access.redhat.com/support) asking for the RHMAP Build Farm URLs that are appropriate for your region.

 ```yaml
frontend:
  builder_android_service_host: "https://androidbuild.feedhenry.com"
  builder_iphone_service_host: "https://iosbuild.feedhenry.com"
```

### Running the Playbook to deploy RHMAP Core

To deploy the Core run the following command from the root of this directory:

```
ansible-playbook -i my-inventory-file playbooks/core.yml
```

### Tags

The following tags are supported when deploying RHMAP Core:

| Tag               | Description
| ------------------|:-------------------------------------------------------------------------------------------------|
| preflight_checks  | Can be skipped to avoid running resource checks on target nodes                                  |
| rpm               | Can be skipped to avoid running `yum install` on required packages. Useful during development    |
| deploy            | Create the RHMAP Core project and its applications                                               |
| monitoring        | Creates only the Nagios component and required resources                                         |
| infrastructure    | Creates only the "Infrastructure" components and required resources                              |
| backend           | Creates only the "Backend" components and required resources                                     |
| frontend          | Creates only the "Frontend" components and required resources                                    |
| nagios            | Force Nagios status checks against an existing RHMAP Core                                        |

***

## Deploy RHMAP 1-Node MBaaS

### Prerequisites

1. Nodes must be registered with [Red Hat Subscription Manager](https://access.redhat.com/documentation/en-US/Red_Hat_Subscription_Management/1/html-single/RHSM)

	**See [Registering With Subscription Manager](#registering-with-subscription-manager) to do this with the Ansible Playbook included in this repo**

2. Installation of RHMAP 1-Node MBaaS in production will require the following resources outlined in the table below at a minimum:


	|  Description                               | Parameter name               | Default value           | Fail/warning/recommended             |
	|--------------------------------------------|------------------------------|-------------------------|--------------------------------------|
	| Min number of CPUs                         | `min_required_vCPUS`         | 2                       | fail only in strict_mode             |
	| Min system memory per node (in MB)         | `required_mem_mb_threshold`  | 7000                    | fail only in strict_mode             |
	| Min total free memory of all nodes (in KB) | `warning_kb_value`           | 4000000                 | warning                              |
	| Number of PVs with 50 GB storage           | `required_50_pv`             | 0                       | warning                              |
	| Number of PVs with 25 GB storage           | `required_25_pv`             | 1                       | warning                              |
	| Number of PVs with  5 GB storage           | `required_5_pv`              | 0                       | warning                              |
	| Number of PVs with  1 GB storage           | `required_1_pv`              | 1                       | warning                              |

### Setting variables

The variables required for installation of RHMAP MBaaS are set in `roles/deploy-mbaas/defaults/main.yml`

Setup the monitoring parameters with SMTP server details. This is required to enable email alerting via Nagios when a monitoring check fails. If you don’t require email alerting or want to set it up at a later time, the sample values can be used.

```yaml
monitoring:
  smtp_server: "localhost"
  smtp_username: "username"
  smtp_password: "password"
  smtp_from_address: "nagios@example.com"
  rhmap_admin_email: "root@localhost"
```

### Running the Playbook to deploy RHMAP 1-Node MBaaS

To deploy the single node persistent MBaaS, create an inventory file based on the templates in `inventory-templates`. Run the following command from the root of this directory:

```
ansible-playbook -i my-inventory-file playbooks/1-node-mbaas.yml`
```

### Tags

The following tags are supported when deploying RHMAP 1-Node MBaaS:

| Tag               | Description
| ------------------|:-------------------------------------------------------------------------------------------------|
| preflight_checks  | Can be skipped to avoid running resource checks on target nodes                                  |
| rpm               | Can be skipped to avoid running `yum install` on required packages. Useful during development    |
| deploy            | Create the RHMAP MBaaS project and its applications                                              |
| nagios            | Force Nagios status checks against an existing RHMAP MBaaS                                       |

***

## Deploy RHMAP 3-Node MBaaS

#### Prerequisites

1. Nodes must be registered with [Red Hat Subscription Manager](https://access.redhat.com/documentation/en-US/Red_Hat_Subscription_Management/1/html-single/RHSM)

	**See [Registering With Subscription Manager](#registering-with-subscription-manager) to do this with the Ansible Playbook included in this repo**

1. You must have at least 3 hosts in the inventory under `mbaas` group.
1. MBaas Nodes should be labelled correctly for high availability. See the [RHMAP Documentation](https://access.redhat.com/documentation/en-us/red_hat_mobile_application_platform/4.3/html-single/mbaas_administration_and_installation_guide/#labelling-for-mbaas-components)
1. Installation of RHMAP 1-Node MBaaS in production will require the following resources outlined in the table below at a minimum:


	Prerequisites:

	|  Description                               | Parameter name               | Default value           | Fail/warning/recommended             |
	|--------------------------------------------|------------------------------|-------------------------|--------------------------------------|
	| Min number of CPUs                         | `min_required_vCPUS`         | 2                       | fail only in strict_mode             |
	| Min system memory per node (in MB)         | `required_mem_mb_threshold`  | 7000                    | fail only in strict_mode             |
	| Min total free memory of all nodes (in KB) | `warning_kb_value`           | 4000000                 | warning                              |
	| Number of PVs with 50 GB storage           | `required_50_pv`             | 3                       | warning                              |
	| Number of PVs with 25 GB storage           | `required_25_pv`             | 0                       | warning                              |
	| Number of PVs with  5 GB storage           | `required_5_pv`              | 0                       | warning                              |
	| Number of PVs with  1 GB storage           | `required_1_pv`              | 1                       | warning                              |


#### Setting variables

The variables required for installation of RHMAP MBaaS are set in `roles/deploy-mbaas/defaults/main.yml`

Setup the monitoring parameters with SMTP server details. This is required to enable email alerting via Nagios when a monitoring check fails. If you don’t require email alerting or want to set it up at a later time, the sample values can be used.

```yaml
monitoring:
  smtp_server: "localhost"
  smtp_username: "username"
  smtp_password: "password"
  smtp_from_address: "nagios@example.com"
  rhmap_admin_email: "root@localhost"
```

#### Running the Playbook to deploy RHMAP 3-Node MBaaS

To deploy the three node MBaaS, create an inventory file based on the templates in `inventory-templates`. Run the following command from the root of this directory:

```
ansible-playbook -i my-inventory-file playbooks/3-node-mbaas.yml`
```

#### Tags

The following tags are supported when deploying RHMAP 3-Node MBaaS:

| Tag               | Description
| ------------------|:-------------------------------------------------------------------------------------------------|
| preflight_checks  | Can be skipped to avoid running resource checks on target nodes                                  |
| rpm               | Can be skipped to avoid running `yum install` on required packages. Useful during development    |
| deploy            | Create the RHMAP MBaaS project and its applications                                              |
| nagios            | Force Nagios status checks against an existing RHMAP MBaaS                                       |


***

## Deploy RHMAP Self-Managed Buildfarm

### Prerequisites

1. ssh access to the master and nodes

### Setting variables

* The variables required for the role `configure-buildfarm` are set in `roles/digger-installer/configure-buildfarm/defaults/main.yml`

```yaml
project_name: jenkins
buildfarm_version: "2.0"

jenkins_home: /var/lib/jenkins

concurrent_android_builds: 10

# List of plugins required and their dependencies. Note, we don't need to support dependencies for
# kubernetes plugin since these will be packaged

jenkins_plugins:

  -
    name: "managed-scripts"
    version: 1.3
    archive: "https://updates.jenkins-ci.org/download/plugins/managed-scripts/1.3/managed-scripts.hpi"
...
```


* The variables required for the role `deploy-jenkins` are set in `roles/digger-installer/deploy-jenkins/defaults/main.yml`

```yaml
project_name: jenkins
enable_oauth: false
master_memory_limit: 3Gi
master_volume_capacity: 40Gi
```

* The variables required for the role `deploy-nagios` are set in `roles/digger-installer/deploy-nagios/defaults/main.yml`

Setup the monitoring parameters with SMTP server details. This is required to enable email alerting via Nagios when a monitoring check fails. If you don’t require email alerting or want to set it up at a later time, the sample values can be used.

```yaml
smtp_server: localhost
smtp_username: username
smtp_password: password
smtp_from_address: admin@example.com
rhmap_admin_email: root@localhost
```

* The variables required for the role `provision-osx` are set in `roles/digger-installer/provision-osx/defaults/main.yml`


### Running the Playbook to deploy RHMAP Buildfarm

* Run  `ansible-playbook -i my-inventory-file playbooks/buildfarm.yml -e "core_project_name=installed-core`

### Tags

The following tags are supported when deploying RHMAP Buildfarm:

| Tag                 | Description
| --------------------|:-------------------------------------------------------------------------------------------------|
| preflight_checks    | Can be skipped to avoid running resource checks on target nodes                                  |
| gluster-fs          | Tasks related to this tag are only run if gluster_storage is defined in the inventory file       |
| android-sdk         | This tag installs the android-sdk pod and syncs the Android SDK to the underlying PV             |
| deploy-jenkins      | This tag installs the jenkins instance                                                           |
| configure-buildfarm | Configuration required for jenkins, including Kubernetes Pod Template                            |
| configure-millicore | Adds the required configuration to millicore in the provided core_project_name                   |
| provision-osx       | Task related to this tag prvisions and configures any specified node(s) for building iOS applications |
| deploy-nagios       | This tag installs a nagios pod in the project and triggers the checks required |
| print-info          | Prints out urls and credentials related to the Jenkins and Nagios Installations  |

***


## Working With Proxies

When deploying RHMAP Core or RHMAP MBaaS to an OpenShift cluster configured behind some additonal configuration needs to be set. In your inventory file set the following values as appropriate for your OpenShift cluster.


| Variable          | Description
| ------------------|:-------------------------------------------------------------------------------------------------|
| proxy_host        | Hostname or IP address of the proxy server                                                       |
| proxy_port        | Port number of the `proxy_host` set above                                                        |
| proxy_user        | Username for basic authentication if required                                                    |
| proxy_pass        | Password for `proxy_user` set above if required                                                  |
| proxy_url         | http://`proxy-host`:`proxy-port` or http://`proxy_user`:`proxy_pass`@`proxy-host`:`proxy-port`   |
| url_to_check      | A **whitelisted** URL to allow the installation to determine settings above are correct          |

***

## Registering With Subscription Manager

By default, when running the installer, we assume all nodes in the cluster have been registered with Red Hat Subscription Manager. However, if registration of nodes has not been completed, a Playbook is provided to carry out this task and to automatically attach the correct Pool ID for RHMAP.

To enable the pulling of protected Docker images from the registry, we need to configure our credentials for https://access.redhat.com. These should be directly in `roles/register/defaults/main.yml`. Any HTTP proxy settings should be included in the inventory as per the example templates.

Once configured as required run the role using the following command:

```
ansible-playbook -i my-inventory-file playbooks/register.yml
```

***

## Upgrading RHMAP

When upgrading the Core and MBaas, the user will follow the existing upgrade path e.g. 4.2.0 -> 4.3.0 -> 4.4.0. We currently don't support upgrading more than one version at a time. The user can complete the number of necessary upgrades to achieve the version they wish.

To run an upgrade, the user must pass two variables on the command line, these are `project_type`, which must be one of `core, 1-node-mbaas, 3-node-mbaas`. The second required variable is `project_name`, which must be the namespace (OpenShift project name) for the project to be upgraded. Example command:

```
ansible-playbook playbooks/upgrade.yml -i my-inventory-file -e "project_type=core" -e "project_name=my-core-project"
```

**Note** At the end of the install, Nagios checks are run, but it is worth manually ensuring all the latest image versions are running.

***

## Seeding Docker Images

To improve user experience during installation (optional) and upgrades (recommended), the Docker images for specific versions can be pulled onto each of the OpenShift Nodes prior to running the `core.yml`, `1-node-mbaas.yml` and `3-node-mbaas.yml` Playbooks.

Images can be seeded by running the `seed-images.yml` Playbook. The `project_type` variable must be set. Valid values for `project_type` are `core` and `mbaas`. The image version must also be passed using the `rhmap_version` variable. An example command is shown below:

```bash
ansible-playbook -i my-inventory-file playbooks/seed-images.yml -e "project_type=core" -e "rhmap_version=4.3"
```

**Note:** This task will take a substantial amount of time, but can be beneficial pre-installation, particularly on low speed networks.

**Note:** Pulling Docker images requires SSH access to OpenShift Nodes and is therefore only supported for OpenShift Enterprise.

### Dependencies

Running the `seed-images` Playbook requires the [`docker-py`](https://docker-py.readthedocs.io/en/stable/) Python module. If this dependency is unmet on the target host, the Playbook will attempt to install `docker-py` along with the package manager [`pip`](https://github.com/pypa/pip) or via `yum` for RHEL OS.

***

## POC Playbook

This repository contains a playbook for demonstration purposes. It creates Core and a single node MBaaS and links them together using the REST api.

After editing your inventory file to target an existing OpenShift cluster, running the playbook at `playbooks/poc.yml` will give you a Core with an MBaaS target created, allowing you to create projects and deploy applications easily. There are a number of prerequisites that need to be met before running this installation.

### Prerequisites:

1. An OpenShift cluster with a valid subscription for RHMAP and SSH access. See [Registering With Subscription Manager](#registering-with-subscription-manager)
1. The following resources should be met:

	|  Description                               | Parameter name               | Default value           | Fail/warning/recommended             |
	|--------------------------------------------|------------------------------|-------------------------|--------------------------------------|
	| Min number of CPUs                         | `min_required_vCPUS`         | 4                       | fail only in strict_mode             |
	| Min system memory per node (in MB)         | `required_mem_mb_threshold`  | 7000                    | fail only in strict_mode             |
	| Min total free memory of all nodes (in KB) | `warning_kb_value`           | 4000000                 | warning                              |
	| Number of PVs with 50 GB storage           | `required_50_pv`             | 0                       | warning                              |
	| Number of PVs with 25 GB storage           | `required_25_pv`             | 3                       | warning                              |
	| Number of PVs with  5 GB storage           | `required_5_pv`              | 2                       | warning                              |
	| Number of PVs with  1 GB storage           | `required_1_pv`              | 2                       | warning                              |

**Note:**

* `strict_mode` variable is disabled by default for this Playbook.
* `master_url` - variable is **Required** - This parameter is required for setting up the MBaaS Target.

The following additional variables can be set:

* `core_project_name` - Sets the project name for RHMAP Core. Set to avoid conflicts when installing more that one RHMAP POCs on single OpenShift instance.
* `mbaas_project_name` - Sets the project name for the RHMAP 1-Node MBaas. Set to avoid conflicts with multiple POC's.
* `mbaas_target_id` - Sets the mbaas target id when configuring the mbaas target. This is **Required**.




### Deploying The POC Projects

After setting the variables above and ensuring the prerequisites are met run the following:

```
ansible-playbook -i my-inventory-file playbooks/poc.yml -e master_url=https://my.example.com:8443 -e mbaas_target_id=target
```

***

## Running with Docker

This repository comes with a Dockerfile which builds a Docker image with all the requirements baked in. This is based on the [S2I Playbook to image](https://github.com/aweiteka/playbook2image) base image. To use this, we must mount an inventory file as a volume at run time and run the Playbooks contained within this repository. The workflow is as follows:

1. Run the image interactively, mounting the inventory of your choice, and the required SSH key for the cluster to be targeted. An example command is shown below:

```bash
docker run -it \
       -v ~/.ssh/id_rsa:/opt/app-root/src/.ssh/id_rsa:Z \
       -v ${HOME}/Desktop/rhmap-ansible/inventories:/opt/app-root/src/inventories \
       -e ANSIBLE_PRIVATE_KEY_FILE=/opt/app-root/src/.ssh/id_rsa \
	   docker.io/philipgough/rhmap-ansible bash
```

* Now, inside the container, run the following:

```
ansible-playbook -i inventories/my-inventory playbooks/core.yml
```

### Docker With OpenShift Cluster Up

1. Build the image following the steps [here](#running-with-docker)
1. Follow the steps [here](#using-openshift-cluster-up) to get the all-in-one OpenShift cluster running locally
1. Ensure the `oc observe` controller is running to create the required secrets
1. Run the Docker image locally, mounting your development templates inside the container as volumes, example:

```bash
docker run -v ${HOME}/Desktop/rhmap-ansible/inventories:/opt/app-root/src/inventories \
    -v ${HOME}/work/fh-core-openshift-templates/generated:/opt/rhmap/templates/core \
    -v ${HOME}/work/fh-openshift-templates:/opt/rhmap/templates/mbaas \
    -e PLAYBOOK_FILE=/opt/app-root/src/playbooks/poc.yml \
    -e INVENTORY_FILE=/opt/app-root/src/inventory-templates/cluster-up-example \
    -e OPTS="-e core_templates_dir=/opt/rhmap/templates/core -e mbaas_templates_dir=/opt/rhmap/templates/mbaas -e strict_mode=false --tags deploy" \
	docker.io/philipgough/rhmap-ansible
```

***

## Troubleshooting and useful Information

### Using Tags

* The tasks in the Ansible Playbooks are tagged allowing you to run specific sections of the scripts. For example, if you just want to bring up the Nagios Pod, you can run `ansible-playbook playbooks/core.yml --tags monitoring.
* Tags can also be skipped using the `--skip-tags="example-tag"` syntax
* Lists can be supplied to both tag commands using comma seperated values. `--tags="x,y,z"`

### Modifying Variables

Variables can also be passed via the command line or edited directly in YAML. For example, the default project name for RHMAP Core is rhmap-core. This can be passed as a variable by running, for example `ansible-playbook core.yml -e "project_name=core"`

This variable could also be updated directly in the YAML file at `roles/deploy-core/defaults/main.yml`.

Alternatively, the variables can be overridden within a role by creating a new file in the structure of `roles/myExampleRole/vars/main.yml`.

### Options

| Option      | Description                                                                                      | Default |
|-------------|--------------------------------------------------------------------------------------------------|---------|
| strict_mode | If false, ignore errors returned during checking the prerequisites and proceed with installation | true    |


### Configuring the log path

If you see the following warning, it means Ansible cannot create the log file specified in Ansible configuration because of user permissions.

```
[WARNING]: log file at ./ansible.log is not writeable and we cannot create it, aborting
```

You can use a custom path for Ansible logs. In `/opt/rhmap/4.x/rhmap-installer/ansible.cfg`, you can change `log_path` property.
This configuration file is `ansible.cfg` file in the source tree.

You can also use the `ANSIBLE_LOG_PATH` environment variable to achieve the same goal.

```
ANSIBLE_LOG_PATH="/tmp/ansible.log" ansible-playbook foo.yml
```

Note that environment variable has a precedence over the `ansible.cfg` file property.


