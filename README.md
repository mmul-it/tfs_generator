# Terraform role for Ansible

## Azure
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

      $ python3 -m venv --system-site-packages venv-terraform
      ...
      $ source venv-terraform/bin/activate
      (venv-terraform)$

- Install dependencies: once in your environment, you need some dependencies
  (like ansible and azure-cli). We already provides a 'requirements.txt' file
  with all the dependencies, so just run:

      (venv-terraform)$ pip install -r requirements.txt
      ...

  and you're ready to go.

- Login with Azure: this is needed only if you store the Terraform state file
  (tfstate) on an Azure Storage Container. This best practice will be very
  usefull when multiple people works on the same Terraform-provided Azure
  infrastructure, but require to log in into Azure via the azure-cli (already
  installed in your environment in the previous step). So, in your environment
  simply execute this command:

      (venv-terraform)$ az login
      To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code XXXXXXXXX to authenticate.

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

      (venv-terraform)$ az login --service-principal -u <user URL> -p <certificate pem> --tenant <tenant id>
  
  NOTE 2: with Terraform versions >=0.13.x and hashicorp/azurerm provider >=
  2.5.x the login as Service Principal isn't supported anymore. If you use
  those more recent versions (or use the defaults provided by the role) you can
  skip the login as a Service Principal. Everything still works as expected.

  You are now ready to go with Terraform state on Azure Storage Container

### Generate Terraform config files
Once you've compiled the inventory (looks at the defaults/main.yml file for the
available variables), you are ready to generate your Terraform resource files.
Just launch:

    (venv-terraform)$ ansible-playbook -v -i inventory/myenv/hosts terraform.yaml
    ...

### Terraform directory structure
Ansible will generate a directory structure based on the *terraform_config_dir*
variable. Here, you can find this subdirectories:

    terraform_config_dir/
    |
    |- bin/          (contains the Terraform binary)
    |- azure-init/   (used to initialize Azure for keeping the tfstate file)
    |- azure/        (this contains your infrastructure resources)

### Prepare Azure for keeping the tfstate file
If it's the first time you use Terraform to provision Azure resources, and you
choosed to keep the Terraform tfstate on Azure, you need to prepare Azure to
store the file. In your ${terraform_config_dir}/azure-init/ directory you've
all the Terraform resources needed to do this.

    (venv-terraform)$ terraform/myenv/bin/terraform init terraform/myenv/azure-init
    ...
    (venv-terraform)$ terraform/myenv/bin/terraform plan terraform/myenv/azure-init
    ...
    (venv-terraform)$ terraform/myenv/bin/terraform apply terraform/myenv/azure-init
    ...

If you starts working on an already managed infrastructure, probably the
tfstate file will be already on Azure, so you can skip this step.
