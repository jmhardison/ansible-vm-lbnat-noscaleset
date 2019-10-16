# Non-ScaleSet VM Deployment in Azure with Custom Image, LB and NAT Rules

This project provides scripts to perform an Ansible based deployment of multiple VM's behind a single load balancer when not using scale-sets and needing individual inbound NAT's in Azure. The expected use is for a scenario involving multiple machines for `users` that are placed behind a load balancer providing a single IP, with a NAT from the public to the inside `TCP/3389` for `RDP` traffic. Many of these functions are achieved in DevTestLabs, however that is not a good solution for all business requirements.

## Setup

This script implies use of the Azure CLI and a properly authenticated system. Use of Service Principals are possible and can be injected to each task using `client_id` `secret` `tenant_id`, however the Azure modules in Ansible do not support associating a NAT Rule to a NIC. As a result, there is at least one step that requires use of the Azure Client to perform that association. To keep consistency, the whole script was changed ot use implied authentication using Azure CLI. There is an optional first step to set the account to deploy to, this can be performed manually by the user or let the script set it to ensure you deploy where intended when you have multiple subscriptions.

* Follow steps [here](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli?view=azure-cli-latest) to ensure your system is setup for use of the Azure CLI, and subsequently perform an `az logon`. This will create the appropriate credentials file in your system for use by this script.

* Follow the appropriate steps to install Ansible on your system with `Python 3.7`, and subsequently execute a `pip install 'ansible[azure]'` command to download and install all the appropriate Azure modules. (Adjust commands accordingly for your OS.)

* Define variables needed in the `var_holder.yml` file.
  * `resource_group_name` - The RG for all components to be deployed into.
  * `name_prefix` - a short prefix used to pre-pend objects created.
  * `region_location` - the desired deployment location
  * `vm_sku_size` - the desired vm tier size for deployment.
  * `vm_base_image_name` - the name of a given custom image.
  * `vm_base_image_resource_group` - the name of the RG that the image is in.
  * `subscription_id` - the target subscription for deployments. Script does not use implied default from AZ CLI.
  * `address_prefixes` - the address space for the vnet
  * `subnet_address_prefixes` - the address space for the first subnet created in the vnet.
  * `admin_user_name` - the desired admin username for vm objects.
  * `nsg_rules_list` - the list of rules to add to the NSG created for a VM object. This can be moved to the playbook if desired.

After all of these are created, you can execute the given scripts.

## Create_Stub_Plumbing

This creates the plumbing infrastructure that is required for the other scripts. It is assumed that you will use both, and subsequently the scripts rely on the naming format defined within this script.

`ansible-playbook create_stub_plumbing.yml`

This will create:

* Resource Group
* Diagnostic Storage Account (Not yet used)
* VNET
* Subnet in VNET
* Public IP (Static) for LB
* Load Balancer

## Create_New_VM

This creates a new VM each run, that uses input variables for uniqueness. This allows you to execute the command, provide a username (or other value that provides that uniqueness) to the script at execution.

Input variables:

* User First Name - this is converted to lowercase for use through the objects. No need to remember upper/lower on removal.
* Home IP - This will be used for the third rule in the NSG associated to each machine. You can remove this from the script if not used.
* Password - This will be used to create the password for the default admin user.

`ansible-playbook create_new_vm.yml`

This will create:

* NIC
* NSG and Rules
* VM Object (with managed disk) from custom image
* Inbound NAT association

This currently uses a AZ CLI method to link the NAT association as the core Ansible Azure modules do not offer this feature yet.
THis currently uses a AZ CLI method to enable boot diagnostics.

## Remove_Existing_VM

This removes an existing VM with each run, that uses input variables for the username to subsequently find the machine and remove it.

It will not remove the last NAT Rule from the load balancer, currently an issue with the core Ansible module. This could be split out to another Azure CLI operation to cleanup.

`ansible-playbook remove_existing_vm.yml`

This will remove for the given username:

* VM Object and Managed Disk
* Inbound NAT association
* NSG
* NIC

This currently uses an AZ CLI method to remove last inbound nat rule when applicable.

## Things you can do

Change your authentication process to use a service principal, and adjust the scripts to use appropriate definition of `client_id` `secret` `tenant_id` and `subscription_id`.
