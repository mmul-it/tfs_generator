# Azure provider with Terraform tf manifests generator role for Ansible

<img src="https://cdn-dynmedia-1.microsoft.com/is/content/microsoftcorp/azure-blue" alt="Azure" width="128px">

## Variables

After defining the Azure provider and its version:

```yaml
terraform_cloud_provider: 'azure'
azure_provider_version: '2.40.0'
```

There are several variables that can be defined to an environment deployed:
general variables and per-host variables:

### General variables

At general level you will need to define all the details for the infrastructure
resources that will be created (check [defaults/main.yml](defaults/main.yml)):

Credentials:

```yaml
# Azure cloud provider default variables

# Azure access credentials
azure_location: ''
azure_subscription_id: ''
azure_client_id: ''
azure_client_secret: ''
azure_tenant_id: ''
```

Shared TF state:

```yaml
# Azure storage blob for synchronized tfstate
azure_tfstate_resource_group_name: "terraform-tfstate"
azure_tfstate_resources:
  - provider
  - resource_group
  - storage
  - storage_container
```

Infrastructure with networks and storages:

```yaml
# Azure infrastructure details
azure_resources:
  - provider
  - resource_group
  - storage
  - additional_storage
  - network
  - network_sec_group
  - availability_set
azure_vpn_resources:
  - virtualnetwork_gw
  - localnetwork_gw
azure_resource_group_name: "azure-resource-group"
azure_network_sec_group_name: 'network_sec_group'
azure_storage_account_name: 'mystorageaccount'
azure_public_ips: []
azure_vnets:
  - name: 'vnet'
    addr_space: '192.168.0.0/24'
    subnets:
      - name: 'subnet_server'
        prefix: '192.168.0.0/24'
    gateway_subnet:
      - name: 'subnet_gateway'
        prefix: '192.168.0.248/29'
        gw_name: 'vnet_gw'
```

The key vault:

```yaml
# Azure key vault
azure_key_vault_enable: false
azure_key_vault_name: 'myazurevault'
azure_key_vault_keyname: 'myazurevaultkey'
```

And then the general VM variables:

```yaml
# Azure VM default variables
azure_vm_admin_username: 'adminuser'
azure_vm_admin_password: '4dm1n!'
azure_vm_disable_password_authentication: true
# Azure VM image to deploy. By default CentOS 7.7 are used, customize it in
# the inventory group or host variables. The list of images is available via
# 'az vm image list' command.
azure_vm_image:
  publisher: 'OpenLogic'
  offer: 'CentOS'
  sku: '7_9-gen2'
  version: '7.9.2020111901'
# Ansible groups to be included for Azure vms_<group-name>.tf file generation
azure_vms_groups:
  - 'all'
```

This rolse supports also Azure's MySQL/MariaDB native databases, that can be
declared as follows (if you need a master/slave setup):

```yaml
# Azure MySQL/MariaDB Databases definition
# (https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/mysql_server)
# (https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/mariadb_server)
azure_database_servers:
  # Master database
  - name: 'master-name'
    kind: 'MariaDB'
    version: '10.2'
    # MySQL/MariaDB common parameters
    admin_user: 'admin'
    admin_password: 'password'
    sku: 'B_Gen5_2'
    storage_mb: 5120
    auto_grow: 'true'
    backup_retention: 7
    geo_redundant_backup: 'false'
    public_access: 'false'
    ssl: 'true'
    # For replicas
    create_mode: 'Default'
    creation_source_id: ''
    restore_point_in_time: ''
    # MySQL specific parameters
    encryption: 'true'
    ssl_minimal_tls: 'TLS1_2'
    # Databases definitions
    databases:
      - name: 'exampledb1'
        charset: 'utf8'
        collation: 'utf8_unicode_ci'
  # Slave database
  - name: 'master-name'
    kind: 'MariaDB'
    version: '10.2'
    # MySQL/MariaDB common parameters
    admin_user: 'admin'
    admin_password: 'password'
    sku: 'B_Gen5_2'
    storage_mb: 5120
    auto_grow: 'true'
    backup_retention: 7
    geo_redundant_backup: 'false'
    public_access: 'false'
    ssl: 'true'
    # For replicas
    create_mode: 'Default'
    creation_source_id: ''
    restore_point_in_time: ''
    # MySQL specific parameters
    encryption: 'true'
    ssl_minimal_tls: 'TLS1_2'
    # Databases definitions
    # For replicas
    create_mode: 'Replica'
    master: 'master-db'
```

### Per host variables

For each VM you'll want to deploy in Azure a
`inventory/lab/host_vars/<ansible-inventory-hostname>.yml` file should be
created like this:

```yaml
---

addresses:
  - ip: 192.168.10.10
    subnet: my_subnet_server
vm_size: Standard_E4ds_v4
osdisk_size: 30
osdisk_encrypt: true

# If Azure start to choose the VM object id will have the uppercase
# Resource-Group-ID value, you must set this variable in order to ignore it,
# otherwise Terraform will always see a change in the VM (this is because M$
# treat the Resource-Group-ID as a case-insensitive value and Terraform work
# it as case-sensitive).
#azure_renamed_id: true
```

This file will be used by the role to generate the Terraform manifest and could
be also used by other ansible roles to act over the created Azure VM.

## Using Azure tools to apply configurations

You can use this role to generate Terraform resource files necessary to
provision your infrastructure on Azure. Also, this role can configure Azure
for you in order to keeps Terraform state online and shared between multiple
users.

### Prerequisites

In order to work with Azure, especially if you need to store the Terraform
state file (tfstate) on your Azure account, you need to perform some actions
before running terraform commands.

- Create a virtual environment: to don't overlap with other installations on
  your computer, it's best to setup a Python virtual environment. To do this
  you need at least python3 and pip installed. Then, go to the repository clone
  root directory and create the environment (if you prefix the name with venv,
  you'll have already the proper exclusion in your .gitignore file):

  ```console
  $ python3 -m venv --system-site-packages venv-terraform
  ...
  $ source venv-terraform/bin/activate
  (venv-terraform)$
  ```

- Install dependencies: once in your environment, you need some dependencies
  (like ansible and azure-cli). We already provides a 'requirements.txt' file
  with all the dependencies, so just run:

  ```
  (venv-terraform)$ pip install -r requirements.txt
  ...
  ```

  and you're ready to go.

- Login with Azure: this is needed only if you store the Terraform state file
  (tfstate) on an Azure Storage Container. This best practice will be very
  usefull when multiple people works on the same Terraform-provided Azure
  infrastructure, but require to log in into Azure via the azure-cli (already
  installed in your environment in the previous step). So, in your environment
  simply execute this command:

  ```
  (venv-terraform)$ az login
  To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code XXXXXXXXX to authenticate.
  ```

  Open your browser in the az provided address then:

  - Enter the code you found in the command output
  - Enter your account Email
  - Enter your account Password
  - If two-factor authentication was enabled on your account, authorize the
    login

  After few seconds the az command on your computer will output a json (which
  contains your account list).

  NOTE 1: if you have configured your login as a Service Principal, you need
  also to login with the user, tenant and certificate provided by your account
  administrator. Once you obtained this, execute:

  ```
  (venv-terraform)$ az login --service-principal -u <user URL> -p <certificate pem> --tenant <tenant id>
  ```

  NOTE 2: with Terraform versions >=0.13.x and hashicorp/azurerm provider >=
  2.5.x the login as Service Principal isn't supported anymore. If you use
  those more recent versions (or use the defaults provided by the role) you can
  skip the login as a Service Principal. Everything still works as expected.

  You are now ready to go with Terraform state on Azure Storage Container

### Generate Terraform config files

Once you've compiled the inventory, you are ready to generate your Terraform
resource files by launching:

```console
(venv-terraform)$ ansible-playbook -v -i inventory/myenv/hosts tests/terraform_tfs_generator.yaml
...
```

### Terraform directory structure

Ansible will generate a directory structure based on the *terraform_config_dir*
variable. Here, you can find this subdirectories:

```console
terraform_config_dir/
|
|- bin/          (contains the Terraform binary)
|- azure-init/   (used to initialize Azure for keeping the tfstate file)
|- azure/        (this contains your infrastructure resources)
```

### Prepare Azure for keeping the tfstate file

If it's the first time you use Terraform to provision Azure resources, and you
choosed to keep the Terraform tfstate on Azure, you need to prepare Azure to
store the file. In your ${terraform_config_dir}/azure-init/ directory you've
all the Terraform resources needed to do this.

```console
(venv-terraform)$ terraform/myenv/bin/terraform init terraform/myenv/azure-init
...
(venv-terraform)$ terraform/myenv/bin/terraform plan terraform/myenv/azure-init
...
(venv-terraform)$ terraform/myenv/bin/terraform apply terraform/myenv/azure-init
...
```

If you start working on an already managed infrastructure, probably the tfstate
file will be already on Azure, so you can skip this step.

### Create the Azure environment

Creating the environment will be a matter of just initializing and using the
`terraform/myenv/azure` manifests, as follows:

```console
(venv-terraform)$ terraform/myenv/bin/terraform init terraform/myenv/azure
...
(venv-terraform)$ terraform/myenv/bin/terraform plan terraform/myenv/azure
...
(venv-terraform)$ terraform/myenv/bin/terraform apply terraform/myenv/azure
...
```
