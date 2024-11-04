# Generate tf manifests for Libvirt provider

<img src="https://libvirt.org/logos/logo-square-128.png" alt="Libvirt" width="128px">

## Variables

After defining the Libvirt provider and its version:

```yaml
tf_cloud_provider: 'libvirt'
tf_libvirt_provider_version: '0.7.1
```

There are several variables that can be defined to an environment deployed:
general variables and per-host variables:

### General variables

At general level you will need to define all the details for the infrastructure
resources that will be created (check
`[defaults/libvirt.yml](defaults/libvirt.yml)`):

You will need a connection url (by default is local qemu system):

```yaml
# Libvirt connection uri
tf_libvirt_uri: 'qemu:///system'
```

The Ansible group of machines that will be part of this deployment:

```yaml
# Which groups of machines will be processed by the role
tf_libvirt_vms_groups:
  - 'mylabvmsgroup'
```

Networks (note that if you don't want the network to start automatically you can
set the `autostart` option to `false`):

```yaml
# Libvirt vNets configuration
tf_libvirt_networks:
  - name: 'mylabnetwork'
    mode: 'nat'
    addresses:
      - '192.168.199.0/24'
```

And finally pools, volumes and cloud-init specific configurations:

```yaml
# Libvirt pools
tf_libvirt_pools:
  - name: 'mylabpool'
    type: 'dir'
    path: '/lab'

# Libvirt volumes
tf_libvirt_volumes:
  - volume_id: 'almalinux-8'
    file: 'almalinux-8.qcow2'
    pool: 'mylabpool'
    source: 'https://repo.almalinux.org/almalinux/8/cloud/x86_64/images/AlmaLinux-8-GenericCloud-latest.x86_64.qcow2'
    format: 'qcow2'

# Libvirt cloud inits
tf_libvirt_cloud_inits:
  - name: 'mylabcloudinit'
    pool: 'mylabpool'
    cfg:
      ssh_pwauth: true
      users:
        - name: myuser
          passwd: "{{ 'password' | password_hash('sha512') }}"
          lock_passwd: false
          sudo: ALL=(ALL) NOPASSWD:ALL
```

In this case a user named `myuser` with password `password` will be created on
each VM.

### Per host variables

For each VM you'll want to deploy in Libvirt a
`inventory/host_vars/<ansible-inventory-hostname>.yml` file should be
created like this (check [tests/inventory/host_vars/myvm-1.yml](tests/inventory/host_vars/myvm-1.yml)):

```yaml
---

tf_libvirt:
  name: myvm-1
  memory: 1024
  vcpu: 1
  cpu_mode: 'host-passthrough'
  cloudinit_disk: 'mylabcloudinit'
  network_interfaces:
    - network_id: mylabnetwork
      addresses: ["192.168.199.31"]
  disks:
    - volume_id: myvm-1-qcow2
      file: 'myvm-1.qcow2'
      format: 'qcow2'
      pool: 'mylabpool'
      size: '10 GB'
      base_volume_id: 'almalinux-8'
```

This file will be used by the role to generate the Terraform manifest and could
be also used by other ansible roles to act over the created Libvirt VM.

Note the value of the `cpu_mode` variable that makes this VM able to do nested
virtualization, because of the `host-passthrough` value. Omitting the parameter
will keep the default CPU mode.

Other options are supported for the specific virtual machine, it is possible to
specify one or more terminal consoles, by adding:

```yaml
  consoles:
    - type: 'pty'
      target_type: 'serial'
      target_port: '0'
```

or even one or more graphics, by adding:

```yaml
  graphics:
    - type: 'vnc'
      listen_type: 'address'
      autoport: 'true'
```

## Using Terraform binary to deploy the Libvirt infrastructure

You can use this role to generate Terraform resource files necessary to
provision your infrastructure on Libvirt. Also, you use these files to deploy
the environment (i.e. on the local machine).

### Generate Terraform config files

Once you've compiled the inventory, you are ready to generate your Terraform
resource files by launching:

```console
> ansible-playbook -i tests/inventory tests/tfs_generator.yml

PLAY [Create Terraform manifests using tfs_generator Ansible role] ******************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [localhost]

...
...

PLAY RECAP ************************************************************************************************************************************************
localhost                  : ok=15   changed=7    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

This will generate a `tests/tf` directory containing all the generated
manifests.

### Terraform directory structure

Ansible will generate a directory structure based on the `tf_config_dir`
variable. Here, you can find this subdirectories:

```console
tests/tf
├── bin
│   └── terraform
├── cloud_init_mylabcloudinit.cfg
├── cloud_init.tf
├── mylabvmsgroup.vms.tf
├── networks.tf
├── pools.tf
├── private
│   └── variables.source
├── provider.tf
├── terraform.tfstate
├── terraform.tfstate.backup
└── volumes.tf
```

## Using tf cmd binary to generate the environment

if the user who executes the binary have the rights to
use libvirt, then the environment will be created with no pain, first
initializing Terraform:

```console
(venv-ansible) rasca@catastrofe [~/Git/mmul-it/tfs_generator]> tests/tf/bin/terraform -chdir=tests/tf init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/template versions matching "2.2.0"...
- Finding dmacvicar/libvirt versions matching "0.7.1"...
- Installing hashicorp/template v2.2.0...
- Installed hashicorp/template v2.2.0 (self-signed, key ID 34365D9472D7468F)
- Installing dmacvicar/libvirt v0.7.1...
- Installed dmacvicar/libvirt v0.7.1 (self-signed, key ID 96B1FE1A8D4E1EAB)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

And then applying the manifests:

```console
> tests/tf/bin/terraform -chdir=tests/tf apply -auto-approve
libvirt_cloudinit_disk.mylabcloudinit: Creating...
libvirt_pool.mylabpool: Creating...
libvirt_network.mylabnetwork: Creating...
libvirt_pool.mylabpool: Creation complete after 5s [id=5650fba8-eadc-46e0-baaf-3799a4a5d59d]
libvirt_volume.almalinux-8: Creating...
libvirt_cloudinit_disk.mylabcloudinit: Creation complete after 5s [id=/lab/cloud_init_mylabcloudinit.iso;2599eac0-0aad-4bbe-920f-3ccace93c2c6]
libvirt_network.mylabnetwork: Creation complete after 5s [id=0aa6db02-110c-4dde-bffb-8c242efe5617]
libvirt_volume.almalinux-8: Still creating... [10s elapsed]
libvirt_volume.almalinux-8: Still creating... [20s elapsed]
libvirt_volume.almalinux-8: Still creating... [30s elapsed]
libvirt_volume.almalinux-8: Still creating... [40s elapsed]
libvirt_volume.almalinux-8: Still creating... [50s elapsed]
libvirt_volume.almalinux-8: Still creating... [1m0s elapsed]
libvirt_volume.almalinux-8: Still creating... [1m10s elapsed]
libvirt_volume.almalinux-8: Still creating... [1m20s elapsed]
libvirt_volume.almalinux-8: Creation complete after 1m24s [id=/lab/almalinux-8.qcow2]
libvirt_volume.myvm-1-qcow2: Creating...
libvirt_volume.myvm-2-qcow2: Creating...
libvirt_volume.myvm-1-qcow2: Creation complete after 1s [id=/lab/myvm-1.qcow2]
libvirt_domain.myvm-1: Creating...
libvirt_volume.myvm-2-qcow2: Creation complete after 1s [id=/lab/myvm-2.qcow2]
libvirt_domain.myvm-2: Creating...
libvirt_domain.myvm-1: Creation complete after 3s [id=3ef50739-9780-4c1d-8b3b-99072e8450b8]
libvirt_domain.myvm-2: Creation complete after 3s [id=4406bae0-6dc2-4d3b-97d5-e050631ced93]

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.
```

In short the VMs should be available:

```console
> ssh myuser@192.168.199.31 uptime
Warning: Permanently added '192.168.199.31' (ED25519) to the list of known hosts.
myuser@192.168.199.31's password:
 12:03:27 up 1 min,  0 users,  load average: 0.01, 0.01, 0.00
```

In the defaults example file cloud-init is configured to set password to
`password`:

```yaml
...

# Libvirt cloud inits
tf_libvirt_cloud_inits:
  - name: 'mylabcloudinit'
    pool: 'mylabpool'
    cfg:
      ssh_pwauth: true
      users:
        - name: myuser
          passwd: "{{ 'password' | password_hash('sha512') }}"
          lock_passwd: false
          sudo: ALL=(ALL) NOPASSWD:ALL
```

but you can check the [cloud-init documentation](https://cloudinit.readthedocs.io/en/latest/reference/examples.html)
to see how it is possible to change `cfg` contents and so the clod-init
behavior.

Note that for each run of `tfs_generator.yml` playbook a cleanup is made to
ensure that there are no residual `.tf` and `.cfg` files from previous runs.
