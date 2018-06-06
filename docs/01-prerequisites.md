# Prerequisites

## Microsoft Azure

This tutorial leverages the [Azure Cloud Infrastructure](https://azure.microsoft.com/en-us/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://azure.microsoft.com/en-us/offers/ms-azr-0044p/) for $200 in free Azure credits.

> The compute resources required for this tutorial might exceed the Azure free tier.

## Azure CLI

### Install the Azure CLI

Follow the [documentation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) to install and configure the `az` command line utility or you can also use the [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview) from your browser in the Azure portal.

Verify Azure CLI is version 2.0 or higher:

```
az --version | grep azure-cli
```

### Set a Default Subscription, location and resource group

If you are using the `az` command-line tool for the first time you must login to your account first:

```
az login
```

Otherwise set your default subscription:

```
az account set --subscription xxx-xxx-xxx-xxx
```

Create a new Resource Group:

```
az group create -l centralus -n MyResourceGroup
```
> Use the `az account list-locations` command to view additional locations.

Set default values for Resource Group and Location:

```
az configure --defaults location=centralus group=MyResourceGroup
```

Display your resource group:

```
az group list -o table
```

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
