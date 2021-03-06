​	

# **SGX Attestation Infrastructure and Secure Key Caching (SKC) Quick Start Guide**

[[_TOC_]]

## **1. Hardware & OS Requirements**

1. **Three Hosts or VMs**

   a.    Build System

   b.    CSP managed Services 

   c.    Enterprise Managed Services

2. **SGX Enabled Host**

3. **OS Requirements**

   UBUNTU 18.04. SKC Solution is built, installed and tested with root privileges. Please ensure that all the following instructions are executed with root privileges

   **Assumption:**

   CSP and Enterprise side deployment will be done through Ansible-Galaxy role;

## **2. Network Requirements**

1. **Build System**

   Internet access required

2. **CSP Managed Services** 

   Internet access required for SGX Caching Service deployed on CSP system/SGX Compute Node;

3. **Enterprise Managed Services**

   Internet access required for SGX Caching Service deployed on Enterprise system;

4. **SGX Enabled Host**

   Internet access required to access KBS running on Enterprise environment

**Setting Proxy and No Proxy**

```
export http_proxy=http://<proxy-url>:<proxy-port>
export https_proxy=http://<proxy-url>:<proxy-port>
export no_proxy=0.0.0.0,127.0.0.1,localhost,<CSP IP>,<Enterprise IP>, <SGX Compute Node IP>, <KBS system Hostname>
```

**Firewall Settings**

Ensure that all the SKC service ports are accessible with firewall

## **3. Ubuntu Package Requirements**

Access required for the postgresql repo in all systems, through below steps:

**Create the file repository configuration**
```
sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

**Import the repository signing key**
```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

**Update the package lists**
```
apt-get update
```

**On SGX Compute Node only, load the MSR driver, as it is not auto-loaded**
```
cd /root
modprobe msr
```

## **4. Deployment Model**

![deploy_model](./images/isecl_deploy_model.PNG)

* Build + Deployment Machine

* CSP - ISecL Services Machine

* CSP - Physical Server as per supported configurations

* Enterprise - ISecL Services Machine


## **5. Ubuntu System Tools and Utilities**

**Ubuntu System Tools and utils**

```
apt-get install -y git gcc zip wget make python3 python3-yaml python3-pip tar lsof jq nginx curl libssl-dev
ln -s /usr/bin/python3 /usr/bin/python
ln -s /usr/bin/pip3 /usr/bin/pip
apt-get update
apt-get install makeself

Install latest libkmip for KBS
git clone https://github.com/openkmip/libkmip.git
cd libkmip
make install

```

***Repo Tool***

```
tmpdir=$(mktemp -d)
git clone https://gerrit.googlesource.com/git-repo $tmpdir
install -m 755 $tmpdir/repo /usr/local/bin
rm -rf $tmpdir
```

***Golang Installation***

```
wget https://dl.google.com/go/go1.14.1.linux-amd64.tar.gz
tar -xzf go1.14.1.linux-amd64.tar.gz
sudo mv go /usr/local
export GOROOT=/usr/local/go
export PATH=$GOROOT/bin:$PATH
rm -rf go1.14.1.linux-amd64.tar.gz
```


## **6. Build Services, Libraries and Install packages**

Note: currently, the repos contain the source code of both the SGX Attestation Infrastructure and SKC. Make will build and package all the binaries and installation scripts  but the SGX Attestation Infrastructure can be installed and deployed separately. SKC cannot be installed without the SGX Attestation Infrastructure. 

The rest of this document will indicate steps that are only needed for SKC. 

**Pulling Source Code**

```
mkdir -p /root/workspace && cd /root/workspace
repo init -u https://github.com/intel-secl/build-manifest.git -b refs/tags/v3.3.1 -m manifest/skc.xml
repo sync
```

**Install, Enable and start the Docker daemon**

  ```shell
  wget https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/amd64/containerd.io_1.2.10-3_amd64.deb 
  dpkg -i containerd.io_1.2.10-3_amd64.deb
  wget "https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/amd64/docker-ce-cli_19.03.5~3-0~ubuntu-bionic_amd64.deb"
  dpkg -i docker-ce-cli_19.03.5~3-0~ubuntu-bionic_amd64.deb
  wget "https://download.docker.com/linux/ubuntu/dists/bionic/pool/stable/amd64/docker-ce_19.03.5~3-0~ubuntu-bionic_amd64.deb"
  dpkg -i docker-ce_19.03.5~3-0~ubuntu-bionic_amd64.deb

  systemctl enable docker
  systemctl start docker

  ```

**Ignore the below steps if not running behind a proxy**

  ```shell
  mkdir -p /etc/systemd/system/docker.service.d
  touch /etc/systemd/system/docker.service.d/proxy.conf
  
  #Add the below lines in proxy.conf and set proxy server details if proxy is used
  [Service]
  Environment="HTTP_PROXY=<http_proxy>"
  Environment="HTTPS_PROXY=<https_proxy>"
  Environment="NO_PROXY=<no_proxy>"
  ```

  ```shell
  #Reload docker
  systemctl daemon-reload
  systemctl restart docker
  ```
**Building All SKC Components**

```
make ubuntu

```

**Copy Binaries to a clean folder**

```
For CSP/Enterprise Deployment Model, copy the generated binaries directory to the /root directory on the CSP/Enterprise system
For Single system model, copy the generated binaries directory to the /root directory on the deployment system
```

## **7. Deployment & Usecase Workflow Tools Installation**

The below installation is required on the Build & Deployment system only and the Platform(Windows,Linux or MacOS) for Usecase Workflow Tool Installation

**Deployment Tools Installation**

* Install Ansible on Build Machine

  ```shell
  pip3 install ansible==2.9.10
  ```

* Install `sshpass` for ansible to connect to remote hosts using SSH

  ```shell
  apt-get install sshpass
  ```

* Create directory for ansible default configuration and hosts file

  ```shell
  mkdir -p /etc/ansible/
  touch /etc/ansible/ansible.cfg
  ```

* Copy the `ansible.cfg` contents from https://raw.githubusercontent.com/ansible/ansible/v2.9.10/examples/ansible.cfg and paste it under `/etc/ansible/ansible.cfg`


### Usecases Workflow Tools Installation

* Postman client should be [downloaded](https://www.postman.com/downloads/) on supported platforms or on the web to get started with the usecase collections.

  > **Note:** The Postman API Network will always have the latest released version of the API Collections. For all releases, refer the github repository for [API Collections](https://github.com/intel-secl/utils/tree/v3.3.1/develop/tools/api-collections)


## **8. Deployment**

The below details would enable the deployment through Ansible Role for Intel® SecL-DC Secure Key Caching Usecase. However the services can still be installed manually using the Product Guide. More details on Ansible Role for Intel® SecL-DC in [Ansible-Role](https://github.com/intel-secl/utils/tree/v3.3.1/develop/tools/ansible-role) repository.

### Download the Ansible Role 

The role can be cloned locally from git and the contents can be copied to the roles folder used by your ansible server 

```shell
#Create directory for using ansible deployment
mkdir -p /root/intel-secl/deploy/

#Clone the repository
cd /root/intel-secl/deploy/ && git clone https://github.com/intel-secl/utils.git

#Checkout to specific release version
cd utils/
git checkout <release-version of choice>
cd tools/ansible-role

#Update ansible.cfg roles_path to point to path(/root/intel-secl/deploy/utils/tools/)
```

### Update Ansible Inventory

The following inventory can be used and created under `/etc/ansible/hosts`.

```
[CSP]
<machine1_ip/hostname>

[Enterprise]
<machine2_ip/hostname>

[Node]
<machine3_ip/hostname>

[CSP:vars]
isecl_role=csp
ansible_user=<ubuntu_user>
ansible_password=<password>

[Enterprise:vars]
isecl_role=enterprise
ansible_user=<ubuntu_user>
ansible_password=<password>

[Node:vars]
isecl_role=node
ansible_user=<ubuntu_user>
ansible_password=<password>
```

> **Note:** Ansible requires `ssh` and `root` user access to remote machines. The following command can be used to ensure ansible can connect to remote machines with host key check `
  ```shell
  ssh-keyscan -H <ip_address> >> /root/.ssh/known_hosts
  ```

### Create and Run Playbook

The following are playbook and CLI example for deploying Intel® SecL-DC binaries based on the supported deployment models and usecases. The below example playbooks can be created as `site-bin-isecl.yml`

> **Note:** If running behind a proxy, update the proxy variables under `vars/main.yml` and run as below

> **Note:** Go through the `Additional Examples and Tips` section for specific workflow samples

**Option 1**

```yaml
- hosts: all
  gather_facts: yes
  any_errors_fatal: true
  vars:
    setup: <setup var from supported usecases>
    binaries_path: <path where built binaries are copied to>
  roles:   
  - ansible-role
  environment:
    http_proxy: "{{http_proxy}}"
    https_proxy: "{{https_proxy}}"
    no_proxy: "{{no_proxy}}"```
```

and

```shell
ansible-playbook <playbook-name>
```

> **Note:** Update the `roles_path` under `ansible.cfg` to point to the cloned repository so that the role can be read by Ansible

OR

**Option 2:**

```yaml
- hosts: all
  gather_facts: yes
  any_errors_fatal: true
  roles:   
  - ansible-role
  environment:
    http_proxy: "{{http_proxy}}"
    https_proxy: "{{https_proxy}}"
    no_proxy: "{{no_proxy}}"
```

and

```shell
ansible-playbook <playbook-name> --extra-vars setup=<setup var from supported usecases> --extra-vars binaries_path=<path where built binaries are copied to> --extra-vars "ansible_sudo_pass=<password>"
```

> **Note:** Update the `roles_path` under `ansible.cfg` to point to the cloned repository so that the role can be read by Ansible


### Additional Examples & Tips

* For `secure-key-caching` usecase following options can be provided during runtime in the playbook for providing the PCS server key

  ```shell
   ansible-playbook <playbook-name> --extra-vars setup=<setup var from supported usecases> --extra-vars binaries_path=<path where built binaries are copied to> --extra-vars intel_provisioning_server_api_key=<pcs server key> --extra-vars "ansible_sudo_pass=<password>"
  ```

  or 

  Update the following vars in `defaults/main.yml`

  ```yaml
  intel_provisioning_server_api_key_sandbox: <pcs server key>
  ```

* If any service installation fails due to any misconfiguration, just uninstall the specific service manually , fix the misconfiguration in ansible and rerun the playbook. The successfully installed services wont be reinstalled.


### Usecase Setup Options

| Usecase                      | Variable                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| Secure Key Caching           | `setup: secure-key-caching` in playbook or via `--extra-vars` as `setup=secure-key-caching`in CLI |

## **9. Usecase Workflows with Postman API Collections**

The below allow to get started with workflows within Intel® SecL-DC for Foundational and Workload Security Usecases. More details available in [API Collections](https://github.com/intel-secl/utils/tree/v3.3.1/develop/tools/api-collections) repository

### Use Case Collections

| Use case                     | Sub-Usecase | API Collection     |
| ---------------------------- | ----------- | ------------------ |
| Secure Key Caching           | -           | ✔️                  |


### Download Postman API Collections

* Postman API Network for latest released collections: https://explore.postman.com/intelsecldc

  or 

* Github repo for allreleases

  ```shell
  #Clone the github repo for api-collections
  git clone https://github.com/intel-secl/utils.git
  
  #Switch to specific release-version of choice
  cd utils/
  git checkout <release-version of choice>
  
  #Import Collections from
  cd tools/api-collections
  ```

>  **Note:**  The postman-collections are also available when cloning the repos via build manifest under `utils/tools/api-collections`


### Running API Collections

* Import the collection into Postman API Client

  > **Note:** This step is required only when not using Postman API Network and downloading from Github

  ![importing-collection](./images/importing_collection.gif)

* Update env as per the deployment details for specific usecase

  ![updating-env](./images/updating_env.gif)

* View Documentation

  ![view-docs](./images/view_documentation.gif)

* Run the workflow

  ![running-collection](./images/running_collection.gif)


## **10. Deployment Using Binaries**

#### Deploying SKC Services on Single System
```
Copy the binaries directory generated in the build system to the /root/ directory on the deployment system
Update skc.conf with the following
  - Deployment system IP address
  - Comment out the K8S_IP,OPENSTACK_IP and TENANT
  - System IP address where Kubernetes or Openstack is deployed
  - Database name, Database username and passwords for AAS, SCS and SHVS services
  - Intel PCS Server API URL and API Keys
Save and Close
./install_skc.sh
```

#### Deploy CSP SKC Services
```
Copy the binaries directory generated in the build system system to the /root/ directory on the CSP system
Update csp_skc.conf with the following
  - CSP system IP Address
  - Database name, Database username and passwords for AAS, SCS and SHVS services
  - Intel PCS Server API URL and API Keys
  - Comment out the K8S_IP,OPENSTACK_IP and TENANT
Save and Close
./install_csp_skc.sh
```

#### Deploy Enterprise SKC Services
```
Copy the binaries directory generated in the build system to the /root/ directory on Enterprise system
Update enterprise_skc.conf with the following
  - Enterprise system IP address
  - Database name, Database username and passwords for AAS and SCS services
  - Intel PCS Server API URL and API Keys
Save and Close
./install_enterprise_skc.sh
```

#### Deploy SGX Agent
```
Copy sgx_agent.tar, sgx_agent.sha2 and agent_untar.sh from binaries directoy to a directory in SGX compute node
./agent_untar.sh
Edit agent.conf with the following
  - CSP system IP address where CMS/AAS/SHVS services deployed
  - CMS TLS SHA Value (Run "cms tlscertsha384" on CSP system)
  - For Each Agent installation on a SGX compute node, please change AGENT_USER (Changing AGENT_PASSWORD is optional)
Save and Close
./deploy_sgx_agent.sh
```

#### Deploy SKC Library
```
Copy skc_library.tar, skc_library.sha2 and skclib_untar.sh from binaries directoy to a directory in SGX compute node
./skclib_untar.sh
Update skc_library.conf with the following
  - IP address for CMS/AAS/KBS services deployed on Enterprise system
  - CSP_CMS_IP should point to the IP of CMS service deployed on CSP system
  - CSP system IP address where SGX Caching Service deployed
  - Hostname of the Enterprise system where KBS is deployed
Save and Close
./deploy_skc_library.sh
```

## **11. System User Configuration**

**Build System**

**Setup ~/.gitconfig to update the git user details. A sample config is provided below**

GIT Configuration**

```
[user]
        name = John Doe
        email = john.doe@abc.com
[color]
        ui = auto
 [push]
        default = matching 
```

## Appendix

### Creating RSA Keys in Key Broker Service

**Configuration Update to create Keys in KBS**

	cd into /root/binaries/kbs_script folder
	
	Update KBS and AAS IP addresses in run.sh
	
	Update CACERT_PATH variable with trustedca certificate inside directory /etc/kbs/certs/trustedca/<id.pem>. 

**Create RSA Key**

	Execute the command
	
	./run.sh reg

- copy the generated cert file to SGX Compute node where skc_library is deployed. Also make a note of the key id generated

### Configuration for NGINX testing

**Note:** Below mentioned OpenSSL and NGINX configuration updates are provided as patches (nginx.patch and openssl.patch) as part of skc_library deployment script.

**OpenSSL**

In the /etc/ssl/openssl.cnf file, look for the below line:
[ new_oids ]

Just before the line [ new_oids ], add the below section:

openssl_conf = openssl_def

[openssl_def]
engines = engine_section
oid_section = new_oids

[engine_section]
pkcs11 = pkcs11_section

[pkcs11_section]
engine_id = pkcs11
dynamic_path =/usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so
MODULE_PATH =/opt/skc/lib/libpkcs11-api.so
init = 0

**Nginx**

Update nginx configuration file /etc/nginx/nginx.conf with below changes:

ssl_engine pkcs11;

Update the location of certificate with the loaction where it was copied into the skc_library machine. 

ssl_certificate "add absolute path of crt file";

Update the KeyID with the KeyID received when RSA key was generated in KBS

ssl_certificate_key "engine:pkcs11:pkcs11:token=KMS;id=164b41ae-be61-4c7c-a027-4a2ab1e5e4c4;object=RSAKEY;type=private;pin-value=1234";

**SKC Configuration**

 Create keys.txt in /tmp folder. This provides key preloading functionality in skc_library.

  Any number of keys can be added in keys.txt. Each PKCS11 URL should contain different Key IDs which need to be transferred from KBS along with respective object tag for each key id specified

  Sample PKCS11 url is as below
  
  pkcs11:token=KMS;id=164b41ae-be61-4c7c-a027-4a2ab1e5e4c4;object=RSAKEY;type=private;pin-value=1234;
  
  Last PKCS11 url entry in keys.txt should match with the one in nginx.conf

  The keyID should match the keyID of RSA key created in KBS. Other contents should match with nginx.conf. File location should match with preload_keys directive in pkcs11-apimodule.ini; 

  Sample /opt/skc/etc/pkcs11-apimodule.ini file
	
	[core]
	preload_keys=/tmp/keys.txt
	keyagent_conf=/opt/skc/etc/key-agent.ini
	mode=SGX
	debug=true
	
	[SW]
	module=/usr/lib/softhsm/libsofthsm2.so
	
	[SGX]
	module=/opt/intel/cryptoapitoolkit/lib/libp11sgx.so

### KBS key-transfer flow validation

On SGX Compute node, Execute below commands for KBS key-transfer:

```
    pkill nginx
```

Remove any existing pkcs11 token

```
    rm -rf /opt/intel/cryptoapitoolkit/tokens/*
```

Initiate Key transfer from KBS

```
    systemctl restart nginx
```

Establish a tls session with the nginx using the key transferred inside the enclave

```
    wget https://localhost:2443 --no-check-certificate
```
