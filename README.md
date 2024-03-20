# Tf manifests generator Ansible role

This role can be used to install tf tool binary and deploy tf manifest files
(`.tf`) starting from an Ansible inventory, to create a fully atomated and
idempotent Infrastructure as Code scenario.

[![Lint and test the project](https://github.com/mmul-it/tfs_generator/actions/workflows/main.yml/badge.svg)](https://github.com/mmul-it/tfs_generator/actions/workflows/main.yml)
[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-tfs_generator-blue.svg)](https://galaxy.ansible.com/mmul/tfs_generator)

## Role Variables

All the useful variables are commented into the defaults file, the main ones are
these:

```yaml
---

# Install tf cmd binary
tf_binary_install: true
tf_binary_version: '0.14.2'
tf_binary_platform: 'linux_amd64'
tf_binary_url: "https://releases.hashicorp.com/terraform/{{ tf_binary_version }}/\
                terraform_{{ tf_binary_version }}_{{ tf_binary_platform }}.zip"

# Name of the command used to generate tf files
tf_cmd: 'terraform'

# Where to deploy tf resource files
tf_config_dir: 'tfs'

# Delete existing resources before deploying new ones
tf_purge: false
```

To start working with this role you'll need to choose which provider will be
used to generate tf manifests, currently `libvirt` (default) and `azure` are
the two supporterd providers.

```yaml
# Which environment we're going to deploy
tf_cloud_provider: 'libvirt'
```

To get specific configuration options look at the [Libvirt](Libvirt.md) or
[Azure](Azure.md) page:

[<img src="https://libvirt.org/logos/logo-square-128.png" alt="Libvirt" width="128px">](Libvirt.md)
[<img src="https://cdn-dynmedia-1.microsoft.com/is/content/microsoftcorp/azure-blue" alt="Azure" width="128px">](Azure.md)

## Example playbook

To test this role and generate a set of tf manifests, just use the
[tests/tfs_generator.yml](tests/tfs_generator.yml) playbook passing the test
inventory with `-i tests/inventory`:

```console
> ansible-playbook -i tests/inventory tests/tfs_generator.yml

PLAY [Create tf manifests using tfs_generator Ansible role] ******************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [localhost]

...
...

PLAY RECAP ************************************************************************************************************************************************
localhost                  : ok=15   changed=7    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

This will generate a `tests/tf` directory containing all the generated
manifests.

To learn how to use this with the tf tool to automate environment generation
check the [Libvirt](Libvirt.md) page.

## License

MIT

## Author Information

Raoul Scarazzini ([rascasoft](https://github.com/rascasoft))
