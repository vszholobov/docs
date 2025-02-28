# Backing up a VM with Hystax Acura Backup


You can create VM backups automatically and recover them in your cloud infrastructure using [Hystax Acura Backup in {{ yandex-cloud }}](/marketplace/products/hystax/hystax-acura-backup).

A VM with Hystax Acura Backup manages the backup and recovery processes. VM backups are saved to an {{ objstorage-name }} [bucket](../../storage/concepts/bucket.md). Recovery is performed using an auxiliary Hystax Cloud Agent VM. It creates a new VM with a certain RPO (recovery point objective) at a point of time in the past. Backup RTO (recovery time objective) depends on the amount of source data.

To back up and recover a VM using Hystax Acura Backup:

1. [Prepare your cloud](#before-begin).
1. [Create a service account](#create-sa).
1. [Configure network traffic permissions](#network-settings).
1. [Create a bucket](#create-bucket).
1. [Create a VM with Hystax Acura Backup](#create-acura-vm).
1. [Make the VM IP address static](#static-ip).
1. [Set up Hystax Acura Backup](#setup-hystax-acura).
1. [Prepare and install an agent on the VM](#prepare-agent).
1. [Create a VM backup](#start-protection).
1. [Create a disaster recovery plan](#create-recovery-plan).
1. [Run the recovery process](#run-recover).

If you no longer need the resources you created, [delete them](#clear-out).

## Prepare your cloud {#before-begin}

{% include [before-you-begin](../_tutorials_includes/before-you-begin.md) %}

### Required paid resources {#paid-resources}

{% note info %}

Note that both the Hystax Acura Backup infrastructure and all the recovered VMs will be charged and counted against the [quotas]({{ link-console-quotas }}).
* A Hystax Acura Backup VM uses 8 vCPUs, 16 GB of RAM, and a 200-GB disk.
* The auxiliary Hystax Cloud Agent VMs use 2 vCPU cores, 4 GB or RAM, and a 10-GB disk. A single Hystax Acura Cloud Agent VM can serve up to 6 replicated disks at the same time. If there are more than 6 disks, additional Hystax Acura Cloud Agent VMs are created automatically.

For detailed system requirements, see the [Hystax documentation](https://xn--q1ach.xn--p1ai/cdn/TechDocs/Deployment-requirements.pdf).

{% endnote %}

The cost of the resources required to use Hystax Acura Backup includes:
* Fee for VM computing resources (see [{{ compute-full-name }} pricing](../../compute/pricing.md#prices-instance-resources)).
* Fee for VM disks (see [{{ compute-full-name }} pricing](../../compute/pricing.md#prices-storage)).
* Fee for using a dynamic or static external IP address (see [{{ vpc-full-name }} pricing](../../vpc/pricing.md#prices-public-ip)).
* Fee for data storage in a bucket and operations with data (see [{{ objstorage-full-name }} pricing](../../storage/pricing.md)).
* Fee for using Hystax Acura Backup (see [product description](/marketplace/products/hystax/hystax-acura-backup) in {{ marketplace-name }}).


### Create a service account and access keys {#create-sa}

Hystax Acura Backup will run under a [service account](../../iam/concepts/users/service-accounts.md).
1. [Create](../../iam/operations/sa/create.md) a service account named `hystax-acura-account` with the `editor` and `marketplace.meteringAgent` roles. Save the service account ID. You will need it later.
1. [Create](../../iam/operations/authorized-key/create.md) an authorized key for the service account. An authorized key is required to perform operations in {{ yandex-cloud }} as a service account. Save the ID and private key. You will need them later.
1. [Create](../../iam/operations/sa/create-access-key.md) a static access key. A static key is required to access a bucket as a service account. Save the ID and secret key. You will need them later.

### Configure network traffic permissions {#network-settings}

Configure network traffic permissions in the [default security group](../../vpc/concepts/security-groups.md#default-security-group).

[Add](../../vpc/operations/security-group-update.md#add-rule) the following rules to it:

Traffic<br>direction | {{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-rule-description }} | {{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-rule-port-range }} | {{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-rule-protocol }} | {{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-rule-destination }} /<br>{{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-rule-source }} | {{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-rule-cidr-blocks }}
--- | --- | --- | --- | --- | ---
Incoming | `http` | `80` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Incoming | `https` | `443` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Incoming | `https` | `4443` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Incoming | `vmware` | `902` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Incoming | `vmware` | `902` | `{{ ui-key.yacloud.common.label_udp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Incoming | `iSCSI` | `3260` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Incoming | `udp` | `12201` | `{{ ui-key.yacloud.common.label_udp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Incoming | `tcp` | `15000` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Outgoing | `http` | `80` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Outgoing | `https` | `443` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Outgoing | `vmware` | `902` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Outgoing | `vmware` | `902` | `{{ ui-key.yacloud.common.label_udp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Outgoing | `iSCSI` | `3260` | `{{ ui-key.yacloud.common.label_tcp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`
Outgoing | `udp` | `12201` | `{{ ui-key.yacloud.common.label_udp }}` | `{{ ui-key.yacloud.vpc.network.security-groups.forms.value_sg-rule-destination-cidr }}` | `0.0.0.0/0`

Save the security group ID. You will need it later.

{% note info %}

Auxiliary Hystax Cloud Agent VMs are created automatically in the default security group. If you created an individual group for the Hystax Acura Backup VM, [move](../../compute/operations/vm-control/vm-update.md) auxiliary Hystax Cloud Agent VMs to this group once they're created.

{% endnote %}

## Create a bucket {#create-bucket}

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), select the folder you want to create a bucket in.
  1. Select **{{ ui-key.yacloud.iam.folder.dashboard.label_storage }}**.
  1. Click **{{ ui-key.yacloud.storage.buckets.button_create }}**.
  1. On the bucket creation page:
      1. Enter a name for the bucket according to the [naming requirements](../../storage/concepts/bucket.md#naming).
      1. Limit the maximum bucket size, if required.

         {% include [storage-no-max-limit](../../storage/_includes_service/storage-no-max-limit.md) %}

      1. Select the type of [access](../../storage/concepts/bucket.md#bucket-access):
          * **{{ ui-key.yacloud.storage.bucket.settings.field_access-read }}**: `{{ ui-key.yacloud.storage.bucket.settings.access_value_private }}`.
          * **{{ ui-key.yacloud.storage.bucket.settings.field_access-list }}**: `{{ ui-key.yacloud.storage.bucket.settings.access_value_private }}`.
          * **{{ ui-key.yacloud.storage.bucket.settings.field_access-config-read }}**: `{{ ui-key.yacloud.storage.bucket.settings.access_value_private }}`.
      1. Select the [storage class](../../storage/concepts/storage-class.md): `{{ ui-key.yacloud.storage.bucket.settings.class_value_standard }}`.
      1. Click **{{ ui-key.yacloud.storage.buckets.create.button_create }}** to complete the operation.
  1. Save the bucket name. You will need it later.

- API {#api}

  Use the [create](../../storage/api-ref/Bucket/create.md) REST API method for the [Bucket](../../storage/api-ref/Bucket/) resource or the [BucketService/Create](../../storage/api-ref/grpc/Bucket/create.md) gRPC API call.

{% endlist %}

## Create a VM with Hystax Acura Backup {#create-acura-vm}

To create a VM with recommended configuration and a boot disk from the Hystax Acura Backup image:

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), select the [folder](../../resource-manager/concepts/resources-hierarchy.md#folder) to create your VM in.
  1. In the list of services, select **{{ ui-key.yacloud.iam.folder.dashboard.label_compute }}**.
  1. In the left-hand panel, select ![image](../../_assets/console-icons/server.svg) **{{ ui-key.yacloud.compute.switch_instances }}**.
  1. Click **{{ ui-key.yacloud.compute.instances.button_create }}**.
  1. Under **{{ ui-key.yacloud.compute.instances.create.section_image }}**:

      * Go to the **{{ ui-key.yacloud.compute.instances.create.image_value_marketplace }}** tab.
      * Click **{{ ui-key.yacloud.compute.instances.create.button_show-all-marketplace-products }}**.
      * In the list of public images, select [Hystax Acura Backup in {{ yandex-cloud }}](/marketplace/products/hystax/hystax-acura-backup) and click **{{ ui-key.yacloud.marketplace-v2.button_use }}**.

  1. Under **{{ ui-key.yacloud.k8s.node-groups.create.section_allocation-policy }}**, select an [availability zone](../../overview/concepts/geo-scope.md) to place your VM in.

      Save the availability zone ID. You will need it later.

  1. Under **{{ ui-key.yacloud.compute.instances.create.section_storages_ru }}**, enter `200 {{ ui-key.yacloud.common.units.label_gigabyte }}` for boot [disk](../../compute/concepts/disk.md) size.
  1. Under **{{ ui-key.yacloud.compute.instances.create.section_platform }}**, select the `8 vCPU` configuration and `16 {{ ui-key.yacloud.common.units.label_gigabyte }}`.
  1. Under **{{ ui-key.yacloud.compute.instances.create.section_network }}**:

      * In the **{{ ui-key.yacloud.component.compute.network-select.field_subnetwork }}** field, specify the subnet ID in the availability zone of the new VM or select a [cloud network](../../vpc/concepts/network.md#network) from the list.

          * Each network must have at least one [subnet](../../vpc/concepts/network.md#subnet). If there is no subnet, create one by selecting **{{ ui-key.yacloud.component.vpc.network-select.button_create-subnetwork }}**.
          * If you do not have a network, click **{{ ui-key.yacloud.component.vpc.network-select.button_create-network }}** to create one:

              * In the window that opens, enter the network name and select the folder to host the network.
              * (Optional) Select the **{{ ui-key.yacloud.vpc.networks.create.field_is-default }}** option to automatically create subnets in all availability zones.
              * Click **{{ ui-key.yacloud.vpc.networks.create.button_create }}**.

      * If a list of **{{ ui-key.yacloud.component.compute.network-select.field_security-groups }}** is available, select the [security group](../../vpc/concepts/security-groups.md#default-security-group) for which you previously configured network traffic permissions. If this list does not exist, all incoming and outgoing traffic will be enabled for the VM.

  1. Under **{{ ui-key.yacloud.compute.instances.create.section_access }}**, specify the information required to access the VM:

      * In the **{{ ui-key.yacloud.compute.instances.create.field_user }}** field, enter a username, e.g., `yc-user`.
      * {% include [access-ssh-key](../../_includes/compute/create/access-ssh-key.md) %}

  1. Under **{{ ui-key.yacloud.compute.instances.create.section_base }}**, specify the VM name: `hystax-acura-vm`.
  1. Under **{{ ui-key.yacloud.compute.instances.create.section_additional }}**, select the `hystax-acura-account` service account.
  1. Click **{{ ui-key.yacloud.compute.instances.create.button_create }}**.

- CLI {#cli}

  {% include [cli-install](../../_includes/cli-install.md) %}

  {% include [default-catalogue](../../_includes/default-catalogue.md) %}

  Run this command:

  ```bash
  yc compute instance create \
    --name hystax-acura-vm \
    --zone <availability_zone> \
    --cores 8 \
    --memory 16 \
    --network-interface subnet-id=<subnet_ID>,nat-ip-version=ipv4,security-group-ids=<security_group_ID> \
    --create-boot-disk name=hystax-acura-disk,size=200,image-id=<Hystax_Acura_Backup_image_ID> \
    --service-account-id <service_account_ID> \
    --ssh-key <path_to_public_SSH_key_file>
  ```

  Where:
  * `--name`: VM name, e.g., `hystax-acura-vm`.
  * `--zone`: [Availability zone](../../overview/concepts/geo-scope.md), e.g., `{{ region-id }}-a`. Save the availability zone ID. You will need it later.
  * `--cores`: [Number of vCPUs](../../compute/concepts/vm.md) the VM has.
  * `--memory`: VM [RAM size](../../compute/concepts/vm.md).
  * `--network-interface`: VM network interface description:
    * `subnet-id`: ID of the subnet to connect your VM to. You can get the list of subnets using the `yc vpc subnet list` CLI command. Save the subnet ID. You will need it later.
    * `nat-ip-version=ipv4`: Connect a public IP address.
    * `security-group-ids`: Security group. Use this parameter if the group is previously configured. You can get the list of groups using the `yc vpc security-group list` CLI command. If you skip this parameter, the [default security group](../../vpc/concepts/security-groups.md#default-security-group) will be assigned.
  * `--create-boot-disk`: Create a new disk for the VM:
    * `name`: Disk name, e.g., `hystax-acura-disk`.
    * `size`: Disk size.
    * `image-id`: Disk image ID. Use `image_id` from the [product description](/marketplace/products/hystax/hystax-acura-backup) in {{ marketplace-name }}.
  * `--service-account-id`: ID of the [previously created](#create-sa) service account. You can get the list of accounts using the `yc iam service-account list` command.
  * `--ssh-key`: Path to the public SSH key file. The default username for access over SSH is `yc-user`.

- API {#api}

  Use the [create](../../compute/api-ref/Instance/create.md) REST API method for the [Instance](../../compute/api-ref/Instance/) resource or the [InstanceService/Create](../../compute/api-ref/grpc/Instance/create.md) gRPC API call.

{% endlist %}

## Make the VM IP address static {#static-ip}

VMs are created with a public dynamic IP. Since a VM with Hystax Acura Backup may reboot, make the IP static.

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), open the page for the folder you are using.
  1. Select **{{ ui-key.yacloud.iam.folder.dashboard.label_vpc }}**.
  1. Go to the **{{ ui-key.yacloud.vpc.switch_addresses }}** tab.
  1. Click ![image](../../_assets/console-icons/ellipsis.svg) in the row with the address of your Hystax Acura Backup VM.
  1. In the menu that opens, select **{{ ui-key.yacloud.vpc.addresses.button_action-static }}**.
  1. In the window that opens, click **{{ ui-key.yacloud.vpc.addresses.popup-confirm_button_static }}**.
  1. Save the IP. You will need it later.

- CLI {#cli}

  {% include [include](../../_includes/cli-install.md) %}

  {% include [default-catalogue](../../_includes/default-catalogue.md) %}

  1. See the description of the CLI update address attribute command:

      ```bash
      yc vpc address update --help
      ```

  1. Get a list of available addresses:

      ```bash
      yc vpc address list
      ```

      Result:

      ```text
      +----------------------+------+-----------------+----------+------+
      |          ID          | NAME |     ADDRESS     | RESERVED | USED |
      +----------------------+------+-----------------+----------+------+
      | e2l46k8conff******** |      | {{ external-ip-examples.0 }}  | false    | true |
      +----------------------+------+-----------------+----------+------+
      ```

      The `false` value of the `RESERVED` parameter indicates that the IP address with the `e2l46k8conff********` `ID` is dynamic.
  1. Make this address static by using the `--reserved=true` key and IP address `ID`:

      ```bash
      yc vpc address update --reserved=true <IP_address_ID>
      ```

      Result:

      ```text
      id: e2l46k8conff********
      folder_id: b1g7gvsi89m3********
      created_at: "2023-05-23T09:36:46Z"
      external_ipv4_address:
        address: {{ external-ip-examples.0 }}
        zone_id: {{ region-id }}-b
        requirements: {}
      reserved: true
      used: true
      ```

      Now that the `reserved` parameter is `true`, the IP address is static.
  1. Save the IP. You will need it later.

- API {#api}

  Use the [update](../../vpc/api-ref/Address/update.md) REST API method for the [Address](../../vpc/api-ref/Address/index.md) resource or the [AddressService/Update](../../vpc/api-ref/grpc/Address/update.md) gRPC API call.

{% endlist %}

## Set up Hystax Acura Backup {#setup-hystax-acura}

1. Open the Hystax Acura Backup VM page in the [management console]({{ link-console-main }}) and find its public IP address.
1. Enter the Hystax Acura Backup VM public IP address in your browser. The initial setup screen opens.

   {% note info %}

   Booting the Hystax Acura Backup VM for the first time will start an installation process which may take over 20 minutes.

   {% endnote %}

   By default, a Hystax Acura VM has a self-signed certificate installed.

1. On the page that opens, fill out the following fields:
   * **Organization**: Name of your organization.
   * **Admin user login**: Administrator username.
   * **Password**: Administrator password.
   * **Confirm password**: Re-enter the administrator password.
1. Click **Next**.
1. Specify the {{ yandex-cloud }} connection settings:
    * **Service account ID**: The service account ID (obtained when [Creating a service account](#create-sa)).
    * **Key ID**: Service account authorized key ID (obtained when [creating a service account](#create-sa)).
    * **Private key**: Service account's private key (obtained when [Creating a service account](#create-sa)).

      {% note info %}

      {% include [hystax-auth-key-newlines](../_tutorials_includes/hystax-auth-key-newlines.md) %}
 
      {% endnote %}

    * **Default folder ID**: [ID](../../resource-manager/operations/folder/get-id.md) of your folder.
    * **Availability zone**: ID of the availability zone hosting the Hystax Acura Backup VM (obtained when [Creating a VM with Hystax Acura Backup](#create-acura-vm)).
    * **Hystax Service Subnet**: ID of the subnet that the Hystax Acura Backup VM is connected to (obtained when [Creating a VM with Hystax Acura Backup](#create-acura-vm)).
    * **S3 Host**: `{{ s3-storage-host }}`.
    * **S3 Port**: `443`.
    * **Enable HTTPS**: Select the option to enable HTTPS connections.
    * **S3 Access Key ID**: Access key ID (obtained when [Creating a service account](#create-sa)).
    * **S3 Secret Access Key**: Secret key (obtained when [Creating a service account](#create-sa)).
    * **S3 Bucket**: Name of the bucket that stores VM backups (you set it when [Creating a bucket](#create-bucket)).
    * **Hystax Acura Control Panel Public IP**: Replace the value with the Hystax Acura Backup VM's public IP (assigned when [Creating a VM with Hystax Acura Backup](#create-acura-vm)).
    * **Additional parameters**: Advanced settings. Do not edit this field.
1. Click **Next**.

Hystax Acura Backup will automatically check whether it can access your cloud. If all the settings are correct, you can log in to the control panel using the previously set username and password.

## Prepare and install an agent on your VMs {#prepare-agent}

To install an agent on the VMs to create backups of:
1. Open the Hystax Acura Backup control panel. Click the Hystax logo.
1. Click **Download agents** on the left.
1. Select the agent type for the OS you need:
   * VMware
   * Windows
   * Linux
1. Click **Next**.
1. Download and install an agent on the VMs to create backups of:

   {% list tabs group=operating_system %}

   - VMware {#vmware}

     1. In the drop-down list, select a group of VMs to set up agents for, such as `Default`.
     1. Select **New VMware vSphere** and fill out the fields:
        * **Platform Name**: Name of the platform.
        * **Endpoint**: public IP address of the ESXi host where the replication agent will be deployed.
        * **Login**: User login (the user must have the administrator permissions).
        * **Password**: Password.

        Click **Next**.
     1. Click **Download Agent** and wait for the agent to download.
     1. Deploy the downloaded OVA file with the agent in your cluster on the VMs to create backups of.
     1. Start the VMs with the agent.

   - Windows {#windows}

     1. In the drop-down list, select a group of VMs to set up agents for, such as `Default`.
     1. Click **Next**.
     1. Click **Download Agent** and wait for the agent to download.
     1. Unpack the archive and install the agent from the `hwragent.msi` file on the VMs to back up.

   - Linux {#linux}

     1. In the drop-down list, select a group of VMs to set up agents for, such as `Default`.
     1. Select Linux distribution type:
        * **CentOS/RHEL (.rpm package)**: CentOS or Red Hat-based.
        * **Debian/Ubuntu (.deb package)**: Ubuntu or Debian.
     1. Select driver install method:
        * **Pre-built**: Install a driver binary.
        * **DKMS**: Compile as you install.
     1. Click **Next**.
     1. You will get commands for installing the agent to the VM. Run these commands following the instructions for your distribution and installation method.

   {% endlist %}

The VMs will appear in the target group of the Hystax Acura Backup control panel a few minutes after the agent is installed.

## Create a VM backup {#start-protection}

Once the agent is installed on the VMs to protect, they will appear in the list as `Unprotected`.

To enable VM protection:
1. Open the Hystax Acura Backup control panel. Click the Hystax logo.
1. Under **Machines Groups**, deploy an instance group, e.g., `Default`.
1. In the VM list on the right, click ![image](../../_assets/console-icons/ellipsis.svg).
1. In the** Edit replication settings** menu, set up a replication schedule for the instance group by hour, day, or week, or select continuous protection. Under **Volume type**, specify the drive type for VM recovery: `network-hdd`, `network-ssd`, or `network-ssd-nonreplicated`. 
1. In the **Edit retention settings** menu, set the backup retention period. For more information, see the [Hystax documentation](https://xn--q1ach.xn--p1ai/documentation/disaster-recovery-and-cloud-backup/dr_overview.html#edit-replication-schedule).
1. Select **Start Protection**.

VM replication will start. A VM replica will include all the data of the original VM. Therefore, replication can take a long time (depending on the original VM's disk size). The replication status will be displayed in the **Status** column under **Machines Groups**. Once it is complete, the VMs will change their status to `Protected`.

## Create a disaster recovery plan {#create-recovery-plan}

The DR plan includes a VM description and the network settings. The plan determines the VMs to recover to your cloud and the VM configuration, subnet, and IP. You can have a plan generated automatically or create one manually:

{% list tabs group=instructions %}

- Automatically {#auto}

  1. Open the Hystax Acura Backup control panel. Click the Hystax logo.
  1. Check the VMs you need on the list, click **Bulk actions**, and select **Generate DR plan**. You can also generate a plan for an instance group by clicking ![image](../../_assets/console-icons/ellipsis.svg) in the group header.
  1. In the **Name** field, enter `Plan-1`.
  1. In the **Subnets** section on the right, set the parameters of the subnet to run the recovered VMs in:
      * In the **Subnet ID** field, enter the subnet ID.
      * In the **CIDR** field, specify the subnet's [CIDR](../../vpc/concepts/network.md#subnet).
  1. Expand the VM description and edit the **Flavor name** field with parameters of the VM to restore as follows: `<platform>-<cpu>-<ram>-<core_fraction>`. For example, `3-8-16-100`.
      
      Where: 
      * `platform`: VM [platform](../../compute/concepts/vm-platforms.md#standard-platforms), such as `1`, `2`, or `3`. 
      * `cpu`: Number of vCPUs.
      * `ram`: Amount of RAM.
      * `core_fraction`: vCPU performance level.
  1. In the **Port ip** field, enter a new IP address for the VM from the selected subnet.
  1. Click **Save**.

- Manually {#manual}

  1. Open the Hystax Acura Backup control panel. Click the Hystax logo.
  1. Click **Add DR Plan**.
  1. In the **Name** field, enter `Plan-1`.
  1. Under **Devices & Ranks**, click ![image](../../_assets/console-icons/ellipsis.svg). In the menu that opens, click **Add machine**. Select an instance group, such as `Default`. Select the VM to add to the DR plan. Repeat the steps for all VMs to recover.
  1. In the **Subnets** section on the right, set the parameters of the subnet to run the recovered VMs in:  
      * In the **Subnet ID** field, enter the subnet ID.
      * In the **CIDR** field, specify the subnet's [CIDR](../../vpc/concepts/network.md#subnet).
  1. Expand the VM description and edit the **Flavor name** field with parameters of the VM to restore as follows: `<platform>-<cpu>-<ram>-<core_fraction>`. For example, `3-8-16-100`.
      
      Where: 
      * `platform`: VM [platform](../../compute/concepts/vm-platforms.md#standard-platforms), such as `1`, `2`, or `3`. 
      * `cpu`: Number of vCPUs.
      * `ram`: Amount of RAM.
      * `core_fraction`: vCPU performance level.
  1. In the **Port ip** field, enter a new IP address for the VM from the selected subnet.
  1. Click **Save**.

{% endlist %}

{% note warning %}

Make sure a valid IP address is specified for each VM.

{% endnote %}

## Run the recovery process {#run-recover}

For a VM's recovery from a backup, Hystax Acura Backup will create a new VM with Hystax Acura Cloud Agent in your cloud. This VM will perform all operations in the cloud.

To run a VM's recovery from a backup:
1. Open the Hystax Acura Backup control panel. Click the Hystax logo.
1. Under **DR plans**, select the previously created plan. Expand plans and edit as required.
1. Click **Run Recover**.
1. In the **Cloud Site Name** field, enter a name, such as `Cloud-Site-from-Plan-1`.
1. Check that all the required resources are listed under **Final DR plan** and click **Run Recover**.

    The Hystax Acura Backup control panel will display a section named **Cloud Sites**. The VM recovery may take a long time. The recovery status will be displayed in the **Status** column under **Machines**. Wait until it changes to `Running`.

1. Open the [management console]({{ link-console-main }}) and check that all the required resources are successfully restored.

## How to delete the resources you created {#clear-out}

To stop paying for the resources you created:
1. [Delete](../../compute/operations/vm-control/vm-delete.md) the Hystax Acura Backup VM.
1. [Delete](../../compute/operations/vm-control/vm-delete.md) the auxiliary Hystax Cloud Agent VMs.
1. [Delete](../../compute/operations/vm-control/vm-delete.md) the recovered VMs.
1. [Delete the bucket](../../storage/operations/buckets/delete.md).
1. [Delete](../../iam/operations/sa/delete.md) the service account used for Hystax Acura Backup.
1. [Delete](../../vpc/operations/address-delete.md) the public static IP you reserved.
