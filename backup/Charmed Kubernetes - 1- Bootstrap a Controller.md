This section will cover how to deploy a Controller in a bare-metal environment.

For The official documentation is already well-prepared for public cloud environments. If you are not deploying in a bare-metal environment, Please refer to https://ubuntu.com/kubernetes/docs for more information.

## Install Juju

Juju is a tool that helps automate the deployment, configuration, and management of applications in cloud environments. It makes it easier to manage multiple applications and their interactions within the cloud.

```bash
sudo snap install juju --channel=2.9 --classic
```

## Setting up password-less login

Generate SSH keys:

```bash
ssh-keygen
```

Copy the generated SSH keys to the specified address

> Replace "ares@100.64.1.81" with the actual username and IP address. The following steps will continue in the same manner. There will be no further prompts.

```bash
ssh-copy-id ares@100.64.1.81
```

## Create a Cloud

Since we are not generating the nodes to be deployed via a cloud environment or infrastructure API, please enter "manual" here.

> You can give a name to your private cloud. I'll use "ares-homelab" here.

```bash
juju add-cloud

This operation can be applied to both a copy on this client and to the one on a controller.
No current controller was detected and there are no registered controllers on this client: either bootstrap one or register one.
Cloud Types
  lxd
  maas
  manual
  openstack
  vsphere

Select cloud type: manual

Enter a name for your manual cloud: ares-homelab

Enter the ssh connection string for controller, username@<hostname or IP> or <hostname or IP>: ares@100.64.1.81

Cloud "ares-homelab" successfully added to your local client.
```

## Bootstrap a Controller

The Juju controller is used to manage the software deployed through Juju, from deployment to upgrades to day-two operations. One Juju controller can manage multiple projects or workspaces, which in Juju are known as ‘models’.

> You can give a name to your Controller. I'll use "infra-demo" here.

```Bash
juju bootstrap ares-homelab infra-demo --no-gui
```

## Enable Controller HA

In a production environment, it is recommended to deploy the Controller in high-availability (HA) mode, with a suggested number of 3 or 5 nodes. If you are only testing, this step can be skipped.

> Enabling HA requires a minimum of three nodes

```Bash
ssh-copy-id ares@100.64.1.82
ssh-copy-id ares@100.64.1.83
juju add-machine -m controller ssh:ares@100.64.1.82
juju add-machine -m controller ssh:ares@100.64.1.83
juju machines -m controller
juju enable-ha --to=1,2
juju controllers --refresh
```

<!-- ##{"timestamp":1667750400,"custom_url":"charmed-kubernetes-chapter-1-bootstrap-a-controller"}## -->