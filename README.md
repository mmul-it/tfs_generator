# Terraform tf manifests generator role for Ansible

This role can be used to install Terraform binary and deploy Terraform manifest
files (`.tf`) starting from an Ansible inventory, to create a fully atomated and
idempotent Infrastructure as Code scenario.

[![Lint and test the project](https://github.com/mmul-it/terraform_tfs_generator/actions/workflows/main.yml/badge.svg)](https://github.com/mmul-it/terraform_tfs_generator/actions/workflows/main.yml)
[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-terraform_tfs_generator-blue.svg)](https://galaxy.ansible.com/mmul/terraform_tfs_generator)

## Role Variables

All the useful variables are commented into the defaults file, the main ones are
these:

```yaml
---

# Install terraform
terraform_binary_install: true
terraform_binary_version: '0.14.2'
terraform_binary_platform: 'linux_amd64'

# Where to deploy terraform resource files
terraform_config_dir: 'terraform'

# Delete existing resources before deploying new ones
terraform_purge: false
```

To start working with this role you'll need to choose which provider will be
used to generate Terraform manifests, currently `libvirt` (default) and `azure`
are the two supporterd providers.

```yaml
# Which environment we're going to deploy
terraform_cloud_provider: 'libvirt'
```

To get specific configuration options look at the [Libvirt](Libvirt.md) or
[Azure](Azure.md) page.

## Example playbook

To test this role and generate a set of Terraform manifests, just use the
[tests/terraform_tfs_generator.yml](tests/terraform_tfs_generator.yml)
playbook passing the test inventory with `-i tests/inventory`:

```console
> ansible-playbook -i tests/inventory tests/terraform_tfs_generator.yml

PLAY [Create Terraform manifests using terraform_tfs_generator Ansible role] ******************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [localhost]

...
...

PLAY RECAP ************************************************************************************************************************************************
localhost                  : ok=15   changed=7    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

This will generate a `tests/terraform` directory containing all the generated
manifests.

To learn how to use this with Terraform to automate environment generation check
the [Libvirt](Libvirt.md) page.

## License

MIT

## Author Information

Raoul Scarazzini ([rascasoft](https://github.com/rascasoft))
