## Azure Terrafy

A tool to bring your existing Azure resources under the management of Terraform.

## Goal

Azure Terrafy imports the resources that are supported by the [Terraform AzureRM provider](https://github.com/hashicorp/terraform-provider-azurerm) within a resource group, into the Terraform state, and generates the corresponding Terraform configuration. Both the Terraform state and configuration are expected to be consistent with the resources' remote state, i.e., `terraform plan` shows no diff. The user then is able to use Terraform to manage these resources.

## Non Goal

The Terraform configurations generated by `aztfy` is not meant to be comprehensive. This means it doesn't guarantee the infrastruction can be reproduced via the generated configurations. For details, please refer to the [limitation](#limitation).

## Install

### From Release

Precompiled binaries and Window MSI are available at [Releases](https://github.com/Azure/aztfy/releases).

### From Go toolchain

```bash
go install github.com/Azure/aztfy@latest
```

### From Package Manager

#### Homebrew (Linux/macOS)

```bash
brew install aztfy
```

#### dnf (Linux)

Supported versions:

- RHEL 8 (amd64, arm64)
- RHEL 9 (amd64, arm64)

1. Import the Microsoft repository key:

    ```
    rpm --import https://packages.microsoft.com/keys/microsoft.asc
    ```

2. Add `packages-microsoft-com-prod` repository:

    ```
    ver=8 # or 9
    dnf install -y https://packages.microsoft.com/config/rhel/${ver}/packages-microsoft-prod.rpm
    ```

3. Install:

    ```
    dnf install aztfy
    ```

#### apt (Linux)

Supported versions:

- Ubuntu 20.04 (amd64, arm64)
- Ubuntu 22.04 (amd64, arm64)

1. Import the Microsoft repository key:

    ```
    curl -sSL https://packages.microsoft.com/keys/microsoft.asc > /etc/apt/trusted.gpg.d/microsoft.asc
    ```

2. Add `packages-microsoft-com-prod` repository:

    ```
    ver=20.04 # or 22.04
    apt-add-repository https://packages.microsoft.com/ubuntu/${ver}/prod
    ```

3. Install:

    ```
    apt-get install aztfy
    ```

#### AUR (Linux)

```bash
yay -S aztfy
```

## Precondition

There is no special precondtion needed for running `aztfy`, except that you have access to Azure.

Although `aztfy` depends on `terraform`, it is not required to have `terraform` pre-installed and configured in the [`PATH`](https://en.wikipedia.org/wiki/PATH_(variable)) before running `aztfy`. `aztfy` will ensure a `terraform` in the following order: 

- If there is already a `terraform` discovered in the `PATH` whose version `>= v0.12`, then use it
- Otherwise, if there is already a `terraform` installed at the `aztfy` cache directory, then use it
- Otherwise, install the latest `terraform` from Hashicorp's release to the `aztfy` cache directory

(The `aztfy` cache directory is at: "[\<UserCacheDir\>](https://pkg.go.dev/os#UserCacheDir)/aztfy")

## Usage

Follow the [authentication guide](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs#authenticating-to-azure) from the Terraform AzureRM provider to authenticate to Azure.

Then you can go ahead and run `aztfy resource [option] <resource id>` or `aztfy resource-group [option] <resource group name>` to import either a single resource, or a resource group and its including resources.

### Terrafy a Single Resource

`aztfy resource [option] <resource id>` terrafies a single resource by its Azure control plane ID.

E.g.

```shell
aztfy resource /subscriptions/0000/resourceGroups/rg1/providers/Microsoft.Compute/virtualMachines/vm1
```

The command will automatically identify the Terraform resource type (e.g. correctly identifies above resource as `azurerm_linux_virtual_machine`), and import it into state file and generate the Terraform configuration.

> ❗For data plane only resources (e.g. `azurerm_key_vault_certificate`), the resource id is using a pesudo format, as is defined [here](https://github.com/magodo/aztft#pesudo-resource-id).

### Terrafy a Resource Group

`aztfy resource-group [option] <resource group name>` terrafies a resource group and its including resources by its name. Depending on whether `--batch` is used, it can work in either interactive mode or batch mode.

#### Interactive Mode

In interactive mode, `aztfy` list all the resources resides in the specified resource group. For each resource, user is expected to input the Terraform resource address in form of `<resource type>.<resource name>` (e.g. `azurerm_linux_virtual_machine.test`). Users can press `r` to see the possible resource type(s) for the selected import item. In case there is exactly one resource type match for the import item, that resource type will be automatically filled in the text input for the users, with a 💡 line prefix as an indication.

In some cases, there are Azure resources that have no corresponding Terraform resources (e.g. due to lacks of Terraform support), or some resources might be created as a side effect of provisioning another resource (e.g. the OS Disk resource is created automatically when provisioning a VM). In these cases, you can skip these resources without typing anything.

> 💡 Option `--resource-mapping`/`-m` can be used to specify a resource mapping file, either constructed manually or from other runs of `aztfy` (generated in the output directory with name: _.aztfyResourceMapping.json_).

After going through all the resources to be imported, users press `w` to instruct `aztfy` to proceed importing resources into Terraform state and generating the Terraform configuration.

As the last step, `aztfy` will leverage the ARM template to inject dependencies between each resource. This makes the generated Terraform template to be useful.

#### Batch Mode

In batch mode, instead of interactively specifying the mapping from Azurem resource id to the Terraform resource address, users are expected to provide that mapping via the resource mapping file, with the following format:

```json
{
    "<azure resource id1>": "<terraform resource type1>.<terraform resource name>",
    "<azure resource id2>": "<terraform resource type2>.<terraform resource name>",
    ...
}
```

Example:

```json
{
  "/subscriptions/0-0-0-0/resourceGroups/tfy-vm/providers/Microsoft.Network/virtualNetworks/example-network": "azurerm_virtual_network.res-0",
  "/subscriptions/0-0-0-0/resourceGroups/tfy-vm/providers/Microsoft.Compute/virtualMachines/example-machine": "azurerm_linux_virtual_machine.res-1",
  "/subscriptions/0-0-0-0/resourceGroups/tfy-vm/providers/Microsoft.Network/networkInterfaces/example-nic": "azurerm_network_interface.res-2",
  "/subscriptions/0-0-0-0/resourceGroups/tfy-vm/providers/Microsoft.Network/networkInterfaces/example-nic1": "azurerm_network_interface.res-3",
  "/subscriptions/0-0-0-0/resourceGroups/tfy-vm/providers/Microsoft.Network/virtualNetworks/example-network/subnets/internal": "azurerm_subnet.res-4"
}
```

Then the tool will import each specified resource in the mapping file (if exists) and skip the others.

Especially if the no resource mapping file is specified, `aztfy` will only import the "recognized" resources for you, based on its limited knowledge on the ARM and Terraform resource mappings.

In the batch import mode, users can further specify the `--continue`/`-k` option to make the tool continue even on hitting import error(s) on any resource.

### Remote Backend

By default `aztfy` uses local backend to store the state file. While it is also possible to use [remote backend](https://www.terraform.io/language/settings/backends), via the `--backend-type` and `--backend-config` options.

E.g. to use the [`azurerm` backend](https://www.terraform.io/language/settings/backends/azurerm#azurerm), users can invoke `aztfy` like following:

```shell
aztfy --backend-type=azurerm --backend-config=resource_group_name=<resource group name> --backend-config=storage_account_name=<account name> --backend-config=container_name=<container name> --backend-config=key=terraform.tfstate <importing resource group name>
```

### Import Into Existing Local State

For local backend, `aztfy` will by default ensure the output directory is empty at the very begining. This is to avoid any conflicts happen for existing user files, including the terraform configuration, provider configuration, the state file, etc. As a result, `aztfy` generates a pretty new workspace for users.

One limitation of doing so is users can't import resources to existing state file via `aztfy`. To support this scenario, you can use the `--append` option. This option will make `aztfy` skip the empty guarantee for the output directory. If the output directory is empty, then it has no effect. Otherwise, it will ensure the provider setting (create a file for it if not exists). Then it proceeds the following steps.

This means if the output directory has an active Terraform workspace, i.e. there exists a state file, any resource imported by the `aztfy` will be imported into that state file. Especially, the file generated by `aztfy` in this case will be named differently than normal, where each file will has `.aztfy` suffix before the extension (e.g. `main.aztfy.tf`), to avoid potential file name conflicts. If you run `aztfy --append` multiple times, the generated config in `main.aztfy.tf` will be appended in each run.

## How it Works

`aztfy` leverage [`aztft`](https://github.com/magodo/aztft) to identify the Terraform resource type on its Azure resource ID. Then it runs `terraform import` under the hood to import each resource. Afterwards, it runs [`tfadd`](https://github.com/magodo/tfadd) to generate the Terraform template for each imported resource.

## Demo

[![asciicast](https://asciinema.org/a/475516.svg)](https://asciinema.org/a/475516)

## Limitation

There are several limitations causing `aztfy` can hardly generate reproducible Terraform configurations.

### N:M Model Mappings

Azure resources are modeled differently in AzureRM provider.

For example, the `azurerm_lb_backend_address_pool_address` is actually a property of `azurerm_lb_backend_address_pool` in Azure platform. Whilst in the AzureRM provider, it has its own resource and a synthetic resource ID as `/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/group1/providers/Microsoft.Network/loadBalancers/loadBalancer1/backendAddressPools/backendAddressPool1/addresses/address1`.

Another popular case is that in the AzureRM provider, there are a bunch of "association" resources, e.g. the `azurerm_network_interface_security_group_association`. These "association" resources represent the association relationship between two Terraform resources (in this case they are `azurerm_network_interface` and `azurerm_network_security_group`). They also have some synthetic resource ID, e.g. `/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mygroup1/providers/microsoft.network/networkInterfaces/example|/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/group1/providers/Microsoft.Network/networkSecurityGroups/group1`.

Currently, this tool only works on the assumption that there is 1:1 mapping between Azure resources and the Terraform resources. For those property-like Terraform resources, `aztfy` will just ignore them.

### AzureRM Provider Validation

When generating the Terraform configuration, not all properties of the resource are exported for different reasons.

One reason is because there are flexible cross-property constraints defined in the AzureRM Terraform provider. E.g. `property_a` conflits with `property_b`. This might due to the nature of the API, or might be due to some deprecation process of the provider (e.g. `property_a` is deprecated in favor of `property_b`, but kept for backwards compatibility). These constraints require some properties must be absent in the Terraform configuration, otherwise, the configuration is not a valid and will fail during `terraform validate`.

Another reason is that an Azure resource can be a property of its parent resource (e.g. `azurerm_subnet` can be its own resource, or be a property of `azurerm_virtual_network`). Per Terraform's best practice, users should only use one of the forms, not both. `aztfy` chooses to always generate all the resources, but omit the property in the parent resource that represents the child resource.

## Additional Resources

- [The aztfy Github Page](https://azure.github.io/aztfy): Everything about aztfy, including comparisons with other existing import solutions.
- [Kyle Ruddy's Blog about aztfy](https://www.kmruddy.com/2022/terrafy-existing-azure-resources/): A live use of `aztfy`, explaining the pros and cons.
- [aztft](https://github.com/magodo/aztft): A Go program and library for identifying the correct Terraform AzureRM provider resource type on the Azure resource id.
- [tfadd](https://github.com/magodo/tfadd): A Go program and library for generating Terraform configuration from Terraform state.
