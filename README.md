# Table of content #

* [Kubernetes Jenkins as Code management](#kubernetes-jenkins-as-code-management)
  * [Prerequisites](#prerequisites)
  * [Installation whiptail/newt](#installation-whiptailnewt)
  * [Installation dialog](#installation-dialog)
* [Basic concept](#basic-concept)
  * [Advantages](#advantages)
* [Build slaves](#build-slaves)
* [Configuration](#configuration)
  * [Configure alternative configuration with overlays](#configure-alternative-configuration-with-overlays)
* [How to use](#how-to-use)
  * [k8s-jcasc.sh arguments](#k8s-jcascsh-arguments)
  * [k8s-jcasc.sh commands](#k8s-jcascsh-commands)
  * [Templates](#templates)
    * [Deployment-only Namespaces](#deployment-only-namespaces)
    * [Sub-Templates (cloud-templates)](#sub-templates-cloud-templates)
* [Execution of Scripts](#execution-of-scripts)
* [IP Management](#ip-management)
* [Additional tools](#additional-tools)
  * [k8sfullconfigexport](#k8sfullconfigexport)
* [Screenshots](#screenshots)
  * [Standard k8s-jcasc-management](#standard-k8s-jcasc-management)
  * [Whiptail based k8s-jcasc-management](#whiptail-based-k8s-jcasc-management)
  * [Dialog based k8s-jcasc-management](#dialog-based-k8s-jcasc-management)
* [Helpful links](#helpful-links)

# Kubernetes Jenkins as Code management #

This project offers a template for managing Jenkins instances on Kubernetes with a JobDSL and Jenkins Configuration as Code (JcasC).

To simplify the installation and the project settings, it has a small helper tool `k8s-jcasc.sh`, which can be used in wizard mode or via arguments to
* create new projects for Jenkins administration
* manage secrets
    * encrypt/decrypt secrets for secure commit to a VCS (version control system)
    * apply secrets to Kubernetes
        * for each project while installation or as an update (`applySecrets`)
        * for all known namespaces, that are configured in the `ip_config.cnf` file (applySecretsToAll)
    * store secrets globally for easy administration
    * store secrets per project for more security
* manage the Jenkins instances for a namespace with the project configuration
    * install
        * create namespace if it does not exist
        * install Jenkins
        * install nginx-ingress-controller per namespace (if configured)
        * install loadbalancer and ingress for Jenkins
    * uninstall
        * uninstall Jenkins installation
        * uninstall nginx-ingress-controller per namespace (if configured)
        * uninstall loadbalancer and ingress for Jenkins (other ingress routes will not be changed)
    * upgrade
* version check to see a message in the console if a new version is available

*If you want to use existing persistent volume claims, then you have to create a persistent volume before you install the application.*

*The password for the preconfigured secrets file is `admin`. There is no valid data inside this file! Please change it for your own project!*

As default the system uses encrypted passwords instead of using the password from the `jenkins_helm_values.yaml`.
The default users and passwords are:

- administrator
  - User: admin
  - Pass: admin
  - permissions: all
- project user
  - User: project-user
  - Pass: project
  - permissions: read all and execute build

This can be changed on the `jcasc_config.yaml` file under the `jenkins.securityRealm` section.

## Prerequisites ##

To use this tool, you need to have the following tools installed:

* bash
* for encryption one of (can be configured):
    * gpg
    * openssl
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm 3](https://helm.sh/)

Optional the tool can use `whiptail` (`newt`) or `dialog` for a better workflow.

### Installation whiptail/newt ###

- Debian: `apt-get install whiptail`
- Ubuntu: `apt-get install whiptail`
- Alpine: `apk add newt`
- Arch Linux: `pacman -S whiptail`
- Kali Linux: `apt-get install whiptail`
- CentOS: `yum install newt`
- Fedora: `dnf install newt`
- OS X: `brew install newt`
- Raspbian: `apt-get install whiptail`

### Installation dialog ###

- Debian: `apt-get install dialog`
- Ubuntu: `apt-get install dialog`
- Alpine: `apk add dialog`
- Arch Linux: `pacman -S dialog`
- Kali Linux: `apt-get install dialog`
- CentOS: `yum install dialog`
- Fedora: `dnf install dialog`
- OS X: `brew install dialog`
- Raspbian: `apt-get install dialog`

# Basic concept #

![alt text](docs/images/k8s-mgmt-workflow.png "K8S Workflow")

* A namespace contains one Jenkins instance.
* The namespace is more or less equal to a k8s-mgmt project
    * Projects are stored in a separate repository in a VCS
    * The project contains
        * the Jenkins Helm Chart values.yaml overwrites
        * one JCasC file
* The Jenkins for each namespace will be deployed via k8s-mgmt from the cloned Git repository
* Jenkins loads its main configuration from the project repository (and only from this, which means you can play around and reload configuration directly from the remote URL)
* This main configuration also contains a very simple `seed-job`, which does a scm checkout of a Groovy script to manage jobs and a repository, which contains the job definition.
    * you can use the [jenkins-jobdsl-remote](https://github.com/Ragin-LundF/jenkins-jobdsl-remote) script as such an seed-job manager.

## Advantages ##
By having all things stored in VCS repositories, which are normally backed up, it is possible to recreate every instance in no-time.
It is impossible to misconfigure a Jenkins instance, because the configuration can be reloaded from this remote repository and all configurations are completely versioned.

Also, every develops maybe can have admin access to play around with the Jenkins, because they can not destroy the system permanently with the beloved "I have nothing done..." statement. 

If the K8S cluster or server crashes, it is possible to redeploy everything as it was in minutes, because also the job definition is stored in a VCS repository.

# Build slaves #
The pre-defined slave-containers will not work directly.
Every build slave container needs to set up the jenkins home work directory and jenkins user/group with `uid`/`gid` `1000`.

Also, the build slaves did not need to have any jenkins agent or something else. Only the user/group and the workdir is needed.

To resolve the problem, that build containers directly shut down, simply add an entrypoint with a `tail -f /dev/null`.

You can also create a Jenkins build slave base container and build your own build tools container on top of it.

Example of a jenkins-build-slave-base-container:

```Dockerfile
FROM alpine:3.10

ARG VERSION=1.0.0
LABEL Description="Jenkins Build Slave Base Container" Vendor="K8S_MGMT" Version="${VERSION}"

###### GLIBC for alpine image
# GLIBC-ENVIROMENT
ENV GLIBC_LANG=en_US
ENV GLIBC_VERSION=2.28-r0
ENV LANG=${GLIBC_LANG}.UTF-8
ENV LANGUAGE=${GLIBC_LANG}.UTF-8

# install base packages, that will be used in most containers
RUN apk update && apk -U upgrade -a && \
    apk add --no-cache xz tar zip unzip sudo curl wget bash git git-lfs procps ca-certificates

# GET GLIBC FROM SGERRAND: https://github.com/sgerrand/alpine-pkg-glibc
RUN wget -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
    wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VERSION}/glibc-${GLIBC_VERSION}.apk && \
    wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VERSION}/glibc-bin-${GLIBC_VERSION}.apk && \
    wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/${GLIBC_VERSION}/glibc-i18n-${GLIBC_VERSION}.apk && \
    apk add --no-cache glibc-${GLIBC_VERSION}.apk glibc-bin-${GLIBC_VERSION}.apk glibc-i18n-${GLIBC_VERSION}.apk && \
    rm -f /etc/apk/keys/sgerrand.* && \
    echo "export GLIBC_LANG=${LANG}" > /etc/profile.d/locale.sh && \
    echo "LANG=${LANG}" >> /etc/environment && \
    /usr/glibc-compat/bin/localedef -i ${GLIBC_LANG} -f UTF-8 ${GLIBC_LANG}.UTF-8 && \
    rm *.apk && \
    echo "Installing additional packages... done"

###### Jenkins setup
# Required Jenkins user/group/gid/uid/workdir
ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
ARG AGENT_WORKDIR=/home/${user}/agent

# create jenkins user
RUN addgroup -g ${gid} ${group} && adduser -h /home/${user} -u ${uid} -G ${group} -D ${user}

# create directories and permissions
RUN mkdir /home/${user}/.jenkins && mkdir -p ${AGENT_WORKDIR}

VOLUME /home/${user}/.jenkins
VOLUME ${AGENT_WORKDIR}

WORKDIR /home/${user}

# let the container tail /dev/null, that Kubernetes will not shut down the container directly after startup.
ENTRYPOINT ["tail", "-f", "/dev/null"]
```

A build-slave container for docker can look then like this:

```Dockerfile
FROM jenkins-slave-base
ARG VERSION=1.0.0
LABEL Description="Docker container with Docker for executing docker build and docker push" Vendor="K8S_MGMT" Version="${VERSION}"

# Installing docker
RUN apk update && apk -U upgrade -a && \
    apk add --no-cache docker

# adding jenkins user to docker group
RUN addgroup -S ${user} docker
```

# Configuration #

The system has a basic configuration file to pre-configure some global settings.
This file is located under [config/k8s_jcasc_mgmt.cnf](config/k8s_jcasc_mgmt.cnf).

It is recommended to change the `PROJECTS_BASE_DIRECTORY` to a directory outside of this project.
The `createproject` command will create new projects as subfolders of this directory.
All files and directories under the `PROJECTS_BASE_DIRECTORY' should be passed to a git repository which is backed up.

Then your existing Jenkins projects can be fully recovered from this repository.

## Configure alternative configuration with overlays ##

To use this repository "as-it-is", it is possible to create a `config/k8s_jcasc_custom.cnf` file.

This file can contain the following configuration:

```bash
# Define path to alternative configuration file
K8S_MGMT_ALTERNATIVE_CONFIG_FILE=/my/path/to/my.config

# Is the alternative configuration only a overlay or a full configuration?
# (true = both config files will be used, false = only the alternative config will be used)
K8S_MGMT_WORK_AS_OVERLAY=true
```

The script checks, if this file exists.
If this is the case, it loads this configuration and checks the argument for the path of the alternative config file.

This means, that the `K8S_MGMT_ALTERNATIVE_CONFIG_FILE` key can define, where the alternative of the `k8s_jcasc_mgmt.cnf` is located.
In the `.gitignore` file, this file is set to ignore, to prevent a commit.

With the `K8S_MGMT_WORK_AS_OVERLAY` key it is possible to tell the system to first load the original configuration (`k8s_jcasc_mgmt.cnf`) and then the alternative config, which was defined with the `K8S_MGMT_ALTERNATIVE_CONFIG_FILE` key.
In the new configuration, it is only required to overwrite the options, that has to be changed instead of writing a new file with the complete configuration, which can result in update problems.

# How to use #

The simplest way is to call the script without arguments. Everything else will be asked by the script.

*Hint: Before you install the Jenkins, you have to commit the files of your project directory and to ensure, that the `jasc_config.yaml` file is readable for Jenkins (public)*

```bash
./k8s-jcasc.sh
```

For less selection and more control you can also give some arguments and the command to the script:

```bash
./k8s-jcasc.sh <arguments> <command>
```

The order of the arguments and commands are irrelevant.

## k8s-jcasc.sh arguments ##

It is possible to use multiple arguments.
The following arguments are supported:

| Argument | Description | Example |
| --- | --- | --- |
| `-p=` or `--projectdir=` | Defines the project directory (or project name) of the Jenkins configuration. This directory is a subdirectory of the configured `PROJECTS_BASE_DIRECTORY` | `-p=myproject` or `--projectdir=myproject` |
| `-n=` or `--namespace=` | Defines the target namespace in Kubernetes. It is not used for encrypting or decrypting secrets. | `-n=jenkins-namespace` or `--namespace=jenkins-namespace` |
| `-d=` or `--deploymentname=` | Defines the deployment name, which is relevant only for `install` and `uninstall`. This can also be configured globally for all projects as `JENKINS_MASTER_DEPLOYMENT_NAME` in the config file. | `-d=jenkins-master` or `--deploymentname=jenkins-master` |
| `--nodialog` | If `dialog` is installed, but you dont want to use it | n/a | 

## k8s-jcasc.sh commands ##

Only one command can be used. Multiple commands are *NOT* supported.
The following commands are supported:

| Command | Description |
| --- | --- |
| `install` | Install Jenkins to a Kubernetes namespace (helm install). |
| `uninstall` | Uninstall the Jenkins instance of a Kubernetes namespace (helm uninstall). |
| `upgrade` | Upgrade the Jenkins instance of a Kubernetes namespace (helm upgrade). |
| `encryptsecrets` | Encrypt the secrets (global secrets or project secrets, depending on configuration). |
| `decryptsecrets` | Decrypt the secrets (global secrets or project secrets, depending on configuration). |
| `applysecrets` | Apply the secrets to the Kubernetes namespace (global secrets or project secrets, depending on configuration). |
| `applySecretsToAll` | Apply the secrets to all known namespaces in Kubernetes (global secrets only). |
| `createproject` | Create a new Jenkins project for the configuration and deployment values from the templates. It uses a wizard to ask for relevant data. |
| `createdeploymentonlyproject` | Create a new project without Jenkins for deployments. This can be used to reserve/manage the IP inside the tool and to install a loadbalancer and ingress controller to this namespace. |
| `createJenkinsUserPassword` | Create a new bcryted Jenkins user password with `htpasswd`. You can also use this online site to create a password: https://www.devglan.com/online-tools/bcrypt-hash-generator  |

## Templates ##

At the `templates` directory contains the following:
- `cloud-templates` -> support for subsets for Jenkins Cloud templates
- `jcasc_config.yaml` -> Jenkins Configuration as Code template
- `jenkins_helm_values.yaml` -> Basic Jenkins configuration of the Helm Charts template
- `nginx_ingress_helm_values.yaml` -> Nginx Ingress Controller Helm Chart values template
- `pvc_claim.yaml` -> Template for Persistent Volume Claim
- `secrets.sh` -> Example of secrets.sh script

### Deployment-only Namespaces ###

`jenkins_helm_values.yaml` offers the possibility to add other namespaces for a Jenkins instance, that should deploy.
The default for this section is empty:

```yaml
[...]
k8smanagement:
  rbac:
    # list of additional namespaces that should be deployable (adds RBAC roles to those namespaces)
    additionalNamespaces: {}
```

To let the Jenkins install applications into other namespaces, these namespaces can be added here.
It is not required to add the namespace in which the Jenkins instance is running.
The Helm Chart will then add additional `Roles` and `RoleBindings` to these namespaces for this instance. 

*This is currently a manual process after creating a new project!*

**Example:**
In this example we add the namespaces `myapplication-qa`, `myapplication-preview` and `myapplication-production` to this Jenkins instance.
After this was deployed, Jenkins can now deploy the application into them. 

```yaml
[...]
k8smanagement:
  rbac:
    # list of additional namespaces that should be deployable (adds RBAC roles to those namespaces)
    additionalNamespaces:
      - myapplication-qa
      - myapplication-preview
      - myapplication-production
```

### Sub-Templates (cloud-templates) ###

`k8s-jcasc-management` supports additional sub-templates to create projects with dynamic container configuration.
These templates are located in the `./templates/cloud-templates/` directory.
If this directory does not exist, the `create project` wizard will not ask for other sub-templates.

All files stored there can be selected with the process/menu `create project` and will added to the `jcasc_config.yaml`. 

The file `jcasc_config.yaml` should now have a `##K8S_MGMT_JENKINS_CLOUD_TEMPLATES##` placeholder:

```yaml
  clouds:
    - kubernetes:
        name: "jenkins-build-slaves"
        serverUrl: ""
        serverCertificate: ##KUBERNETES_SERVER_CERTIFICATE##
        directConnection: false
        skipTlsVerify: true
        namespace: "##NAMESPACE##"
        jenkinsUrl: "http://##JENKINS_MASTER_DEPLOYMENT_NAME##:8080"
        maxRequestsPerHostStr: 64
        retentionTimeout: 5
        connectTimeout: 10
        readTimeout: 20
        templates:
##K8S_MGMT_JENKINS_CLOUD_TEMPLATES##
```

**It is important, that the placeholder is at the beginning of the line.**

If files are inside of this directory, the user can then select which (none or multiple) sub-templates should be added to the main template.

These sub-templates must also start on the beginning of the line.
For an example have a look here: [templates/cloud-templates/node.yaml](./templates/cloud-templates/node.yaml)


# Execution of Scripts #

It is also possible to create shell scripts for namespaces/directories. This can be helpful, if you want to install other tools besides Jenkins.
The `k8s-jcasc.sh` tool first tries to install the secrets, PVC and Jenkins. After this was done it checks, if a directory called `scripts` is inside of the project directory and if it contains `*.sh` files.

These files have to follow these rules and naming conventions:
- `i_*.sh` -> Files that should be executed for installation
- `d_*.sh` -> Files that should be executed for deinstallation

The deinstallation scripts can only be executed if the project directory matches the namespace name. This is necessary because normally only the namespace and no directory selection is required for deinstallation.

# IP Management #

For greater installations and also after a recovery case, it is helpful to know which Jenkins instance is running behind which loadbalancer IP on which namespace.

To provide a simple solution, the system stores these information (namespace and IP) into a configuration file, which can also be backed up.
For every deployment of Jenkins, the system looks into this file and configures the loadbalancer with the IP. This also allows static DNS records.

If you create a new project via the wizard, the system also checks, if a IP address already exists to avoid IP conflicts.

# Additional tools #
## k8sfullconfigexport ##

You can use this tool to export the complete Kubernetes configuration to a local `k8s-manifests` directory.
This can help to figure out differences between clusters.

# Screenshots #

## Standard k8s-jcasc-management ##
![alt text](docs/images/screenshot.png "K8S JCASC Management standard flow")

## Whiptail based k8s-jcasc-management ##
![alt text](docs/images/whiptail_main_menu.png "K8S JCASC Management main menu with whiptail")
*Screenshot: main menu with whiptail support*

![alt text](docs/images/whiptail_directory_select.png "K8S JCASC Management directory selection with whiptail")
*Screenshot: enter a directory with whiptail support*

## Dialog based k8s-jcasc-management ##
![alt text](docs/images/dialog_main_menu.png "K8S JCASC Management main menu with dialog")
*Screenshot: main menu with dialog support*

![alt text](docs/images/dialog_directory_select.png "K8S JCASC Management directory selection with dialog")
*Screenshot: enter a directory with dialog support*



# Helpful links #

- Kubernetes DNS-Based Service Discovery: https://github.com/kubernetes/dns/blob/master/docs/specification.md
- JCasC Examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos
- Jenkins Seed Job script to create jobs from a JSON in a GIT repository: https://github.com/Ragin-LundF/jenkins-jobdsl-remote
- Medium article about the background: https://medium.com/@ragin/jenkins-jenkins-configuration-as-code-jcasc-together-with-jobdsl-on-kubernetes-2f5a173491ab
