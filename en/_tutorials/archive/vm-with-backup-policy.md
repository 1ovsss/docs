# Linking a {{ backup-full-name }} policy to a VM automatically


In {{ yandex-cloud }}, based on a [supported {{ backup-name }} image](../../backup/concepts/vm-connection.md#os), you can create a virtual machine to which a [backup policy](../../backup/concepts/policy.md) will be automatically linked. To do this, you need to provide in the [metadata](../../compute/concepts/vm-metadata.md) the script to install the backup agent and the IDs of the required policies. The specified policies will automatically link to the VM after the VM and agent are created, initialized, and launched.

To create a virtual machine with automatic linking to a {{ backup-name }} policy:

1. [Prepare your cloud](#before-begin).
1. [Activate the service](#service-activate).
1. [Create a service account](#create-sa).
1. [Create a cloud network and subnets](#create-network).
1. [Create and configure a security group](#create-sg).
1. [Create a backup policy](#create-policy).
1. [Create a VM](#create-vm).

If you no longer need the resources you created, [delete them](#clear-out).

## Prepare your cloud {#before-begin}

{% include [before-you-begin](../_tutorials_includes/before-you-begin.md) %}

### Required paid resources {#paid-resources}

The infrastructure support cost includes:

* Fee for VM computing resources (see [{{ compute-full-name }} pricing](../../compute/pricing.md#prices-instance-resources)).
* Fee for VM disks (see [{{ compute-full-name }} pricing](../../compute/pricing.md#prices-storage)).
* Fee for using a dynamic external IP address (see [{{ vpc-full-name }} pricing](../../vpc/pricing.md#prices-public-ip)).
* Fee for VMs connected to {{ backup-name }} and the backup size (see [{{ backup-full-name }} pricing](../../backup/pricing.md#rules)).

## Activate the service {#service-activate}

{% note info %}

The minimum [folder](../../resource-manager/concepts/resources-hierarchy.md#folder) role required to [activate the service](../../backup/concepts/index.md#providers) is `backup.editor` (see [its description](../../backup/security/index.md#backup-editor) for details).

{% endnote %}

To activate the service:

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), select the folder you want to create a VM with a {{ backup-name }} connection in.
  1. In the list of services, select **{{ ui-key.yacloud.iam.folder.dashboard.label_backup }}**.
  1. If you have not activated {{ backup-name }} yet, click **{{ ui-key.yacloud.backup.button_action-activate }}**.

      If there is no **{{ ui-key.yacloud.backup.button_action-activate }}** button, and you have access to creating a VM with a {{ backup-name }} connection, it means the service has already been activated. Proceed to the next step.

{% endlist %}

## Create a service account {#create-sa}

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), select the folder the service is activated in.
  1. At the top of the screen, go to the **{{ ui-key.yacloud.iam.folder.switch_service-accounts }}** tab.
  1. Click **{{ ui-key.yacloud.iam.folder.service-accounts.button_add }}**.
  1. Enter a name for [the service account](../../iam/concepts/users/service-accounts.md): `backup-sa`.
  1. Click ![plus-sign](../../_assets/console-icons/plus.svg) **{{ ui-key.yacloud.iam.folder.service-account.label_add-role }}** and select the `backup.editor` [role](../../backup/security/index.md#backup-editor).
  1. Click **{{ ui-key.yacloud.iam.folder.service-account.popup-robot_button_add }}**.

- {{ yandex-cloud }} CLI {#cli}

  {% include [cli-install](../../_includes/cli-install.md) %}

  {% include [default-catalogue](../../_includes/default-catalogue.md) %}

  1. Create a service account named `backup-sa`.

     ```bash
     yc iam service-account create --name backup-sa
     ```

     Result:

     ```text
     id: ajehb3tcdfa1********
     folder_id: b1g86q4m5vej********
     created_at: "2024-07-22T16:05:14.237381531Z"
     name: backup-sa
     ```

     For more information about the `yc iam service-account create` command, see the [CLI reference](../../cli/cli-ref/managed-services/iam/service-account/create.md).
  
  1. Assign the service account the `backup.editor` role for the folder:

     ```bash
     yc resource-manager folder add-access-binding <folder_ID> \
       --role backup.editor \
       --subject serviceAccount:<service_account_ID>
     ```

     Result:

     ```text
     done (3s)
     effective_deltas:
       - action: ADD
         access_binding:
           role_id: backup.editor
           subject:
             id: ajehb3tcdfa1********
             type: serviceAccount
     ```

     For more information about the `yc resource-manager folder add-access-binding` command, see the [CLI reference](../../cli/cli-ref/managed-services/resource-manager/folder/add-access-binding.md).

- API {#api}

  To create a service account, use the [create](../../iam/api-ref/ServiceAccount/create.md) REST API method for the [ServiceAccount](../../iam/api-ref/ServiceAccount/index.md) resource or the [ServiceAccountService/Create](../../iam/api-ref/grpc/service_account_service.md#Create) gRPC API call.

  To assign the service account the `backup.editor` role for the folder, use the [setAccessBindings](../../iam/api-ref/ServiceAccount/setAccessBindings.md) method for the [ServiceAccount](../../iam/api-ref/ServiceAccount/index.md) resource or the [ServiceAccountService/SetAccessBindings](../../iam/api-ref/grpc/service_account_service.md#SetAccessBindings) gRPC API call.

{% endlist %}

## Create a cloud network and subnets {#create-network}

Create a [cloud network](../../vpc/concepts/network.md#network) with a [subnet](../../vpc/concepts/network.md#subnet) in the [availability zone](../../overview/concepts/geo-scope.md) that will host your VM.

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), select the folder you want to create a cloud network in.
  1. In the list of services, select **{{ ui-key.yacloud.iam.folder.dashboard.label_vpc }}**.
  1. At the top right, click **{{ ui-key.yacloud.vpc.networks.button_create }}**.
  1. In the **{{ ui-key.yacloud.vpc.networks.create.field_name }}** field, specify `cloud-network`.
  1. In the **{{ ui-key.yacloud.vpc.networks.create.field_advanced }}** field, select **{{ ui-key.yacloud.vpc.networks.create.field_is-default }}**.
  1. Click **{{ ui-key.yacloud.vpc.networks.button_create }}**.

- {{ yandex-cloud }} CLI {#cli}

  1. Create a cloud network named `cloud-network`:
  
     ```bash
     yc vpc network create cloud-network
     ```

     Result:

     ```text
     id: enptrcle5q3d********
     folder_id: b1g9hv2loamq********
     created_at: "2024-06-08T09:25:03Z"
     name: cloud-network
     default_security_group_id: enpbsnnop4ak********
     ```

     For more information about the `yc vpc network create` command, see the [CLI reference](../../cli/cli-ref/managed-services/vpc/network/create.md).

  1. Create a subnet named `cloud-network-{{ region-id }}-d` in the `{{ region-id }}-d` availability zone:
  
     ```bash
     yc vpc subnet create cloud-network-{{ region-id }}-d \
       --zone {{ region-id }}-d \
       --network-name cloud-network \
       --range 10.1.0.0/16
     ```

     Result:

     ```text
     id: e9bnnssj8sc8********
     folder_id: b1g9hv2loamq********
     created_at: "2024-06-08T09:27:00Z"
     name: cloud-network-{{ region-id }}-d
     network_id: enptrcle5q3d********
     zone_id: {{ region-id }}-d
     v4_cidr_blocks:
     - 10.1.0.0/16
     ```

     For more information about the `yc vpc subnet create` command, see the [CLI reference](../../cli/cli-ref/managed-services/vpc/subnet/create.md).

- API {#api}

  1. Create a network named `cloud-network` using the [create](../../vpc/api-ref/Network/create.md) REST API method for the [Network](../../vpc/api-ref/Network/index.md) resource or the [NetworkService/Create](../../vpc/api-ref/grpc/network_service.md#Create) gRPC API call.
  1. Create the `cloud-network-{{ region-id }}-d` subnet using the [create](../../vpc/api-ref/Subnet/create.md) REST API method for the [Subnet](../../vpc/api-ref/Subnet/index.md) resource or the [SubnetService/Create](../../vpc/api-ref/grpc/subnet_service.md#Create) gRPC API call.

{% endlist %}

## Create and configure a security group {#create-sg}

For the {{ backup-name }} agent to exchange data with the [backup provider](../../backup/concepts/index.md#providers) servers, the security group must contain the rules that allow network access to the IP addresses of the {{ backup-name }} resources.

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), go to the folder you want to create a VM with a {{ backup-name }} connection in.
  1. In the list of services, select **{{ ui-key.yacloud.iam.folder.dashboard.label_vpc }}**.
  1. In the left-hand panel, select ![image](../../_assets/console-icons/shield.svg) **{{ ui-key.yacloud.vpc.switch_security-groups }}**.
  1. Click **{{ ui-key.yacloud.vpc.network.security-groups.button_create }}**.
  1. In the **{{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-name }}** field, specify `backup-sg`.
  1. In the **{{ ui-key.yacloud.vpc.network.security-groups.forms.field_sg-network }}** field, select `cloud-network`.
  1. Under **{{ ui-key.yacloud.vpc.network.security-groups.forms.label_section-rules }}**, go to the **{{ ui-key.yacloud.vpc.network.security-groups.label_egress }}** tab and click **{{ ui-key.yacloud.vpc.network.security-groups.button_add-rule }}**.
  1. Add the following outgoing traffic rules one by one:

      {% include [outgoing-traffic](../../_includes/backup/outgoing-rules.md) %}

  1. Click **{{ ui-key.yacloud.common.create }}**.

- {{ yandex-cloud }} CLI {#cli}

  Run the following command:

  ```bash
  yc vpc security-group create backup-sg \
    --network-name network-1 \
    --rule direction=egress,port=80,protocol=tcp,v4-cidrs=[213.180.193.0/24] \
    --rule direction=egress,port=80,protocol=tcp,v4-cidrs=[213.180.204.0/24] \
    --rule direction=egress,port=443,protocol=tcp,v4-cidrs=[84.47.172.0/24] \
    --rule direction=egress,port=443,protocol=tcp,v4-cidrs=[84.201.181.0/24] \
    --rule direction=egress,port=443,protocol=tcp,v4-cidrs=[178.176.128.0/24] \
    --rule direction=egress,port=443,protocol=tcp,v4-cidrs=[213.180.193.0/24] \
    --rule direction=egress,port=443,protocol=tcp,v4-cidrs=[213.180.204.0/24] \
    --rule direction=egress,from-port=7770,to-port=7800,protocol=tcp,v4-cidrs=[84.47.172.0/24] \
    --rule direction=egress,port=8443,protocol=tcp,v4-cidrs=[84.47.172.0/24] \
    --rule direction=egress,port=44445,protocol=tcp,v4-cidrs=[51.250.1.0/24] \
    --rule direction=ingress,port=22,protocol=tcp,v4-cidrs=[0.0.0.0/0]
  ```

  Result:

  ```text
  id: enp0v73fe8fs********
  folder_id: b1g86q4m5vej********
  created_at: "2024-07-22T20:17:43Z"
  name: backup-sgg
  network_id: enp3srbi9u49********
  status: ACTIVE
  rules:
  - id: enpporsovuhj********
      direction: EGRESS
      ports:
        from_port: "80"
        to_port: "80"
      protocol_name: TCP
      protocol_number: "6"
      cidr_blocks:
        v4_cidr_blocks:
          - 213.180.193.0/24
  - id: enp7p6asol5i********
      direction: EGRESS
      ports:
        from_port: "80"
        to_port: "80"
      protocol_name: TCP
      protocol_number: "6"
      cidr_blocks:
        v4_cidr_blocks:
          - 213.180.204.0/24
  ...
  - id: enp36mip5nhe********
      direction: INGRESS
      ports:
        from_port: "22"
        to_port: "22"
      protocol_name: TCP
      protocol_number: "6"
      cidr_blocks:
        v4_cidr_blocks:
          - 0.0.0.0/0
  ```

  For more information about the `yc vpc security-group create` command, see the [CLI reference](../../cli/cli-ref/managed-services/vpc/security-group/create.md).

- API {#api}

  To create a security group, use the [create](../../vpc/api-ref/SecurityGroup/create.md) REST API method for the [SecurityGroup](../../vpc/api-ref/SecurityGroup/index.md) resource or the [SecurityGroupService/Create](../../vpc/api-ref/grpc/security_group_service.md#Create) gRPC API call.

{% endlist %}

## Create a backup policy {#create-policy}

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), select the [folder](../../resource-manager/concepts/resources-hierarchy.md#folder) you want to create a backup policy in.
  1. In the list of services, select **{{ ui-key.yacloud.iam.folder.dashboard.label_backup }}**.
  1. Go to ![policies](../../_assets/console-icons/calendar.svg) **{{ ui-key.yacloud.backup.label_policies }}**.
  1. Click **{{ ui-key.yacloud.backup.button_create-policy }}**.
  1. Specify the policy properties:

     * **{{ ui-key.yacloud.common.name }}**: `daily-backup`.
     * **{{ ui-key.yacloud.backup.field_repeat-period-type }}**: `{{ ui-key.yacloud.backup.value_period-time-weekly }}`.
     * **{{ ui-key.yacloud.backup.field_days-of-week }}**: `{{ ui-key.yacloud.backup.value_day-fri }}`.
     * **{{ ui-key.yacloud.backup.field_time }}**: `03:00`.
     * **{{ ui-key.yacloud.backup.field_backup-scheme }}**: `{{ ui-key.yacloud.backup.value_type-incremental }}`.
     * **{{ ui-key.yacloud.backup.field_auto-delete }}**: `{{ ui-key.yacloud.backup.value_retention-save-all }}`.

  1. Click **{{ ui-key.yacloud.common.save }}**.

- CLI {#cli}

  1. Describe the configuration of the backup policy you are creating in the `backup-policy-schema.json` file.

     ```json
     {
       "compression": "NORMAL",
       "format": "AUTO",
       "multiVolumeSnapshottingEnabled": true,
       "preserveFileSecuritySettings": true,
       "reattempts": {
         "enabled": true,
         "interval": {
           "type": "SECONDS",
           "count": "30"
         },
         "maxAttempts": "30"
       },
       "silentModeEnabled": true,
       "splitting": {
         "size": "1099511627776"
       },
       "vmSnapshotReattempts": {
         "enabled": true,
         "interval": {
           "type": "MINUTES",
           "count": "5"
         },
         "maxAttempts": "3"
       },
       "vss": {
         "enabled": true,
         "provider": "TARGET_SYSTEM_DEFINED"
       },
       "archive": {
         "name": "'[Machine Name]-[Plan ID]-[Unique ID]A'"
       },
       "performanceWindow": {
         "enabled": true
       },
       "scheduling": {
         "backupSets": [
           {
             "time": {
               "weekdays": [
                 "FRIDAY"
               ],
               "repeatAt": [
                 {
                   "hour": "3"
                 }
               ],
               "type": "WEEKLY"
             }
           }
         ],
         "enabled": true,
         "maxParallelBackups": "2",
         "randMaxDelay": {
           "type": "MINUTES",
           "count": "30"
         },
         "scheme": "ALWAYS_INCREMENTAL",
         "weeklyBackupDay": "MONDAY"
       },
       "cbt": "ENABLE_AND_USE",
       "fastBackupEnabled": true,
       "quiesceSnapshottingEnabled": true
     }
     ```

  1. Create a backup policy:

     ```bash
     yc backup policy create \
       --name daily-backup \
       --settings-from-file ./backup-policy-scheme.json
     ```

     Result:

     ```text
     id: cdgo5vytuw57********
     name: daily-backup
     created_at: "2024-07-23T20:34:37Z"
     updated_at: "2024-07-23T20:34:37Z"
     enabled: true
     settings:
       compression: NORMAL
       format: AUTO
       multi_volume_snapshotting_enabled: true
       preserve_file_security_settings: true
       reattempts:
         enabled: true
         interval:
           type: SECONDS
           count: "30"
         max_attempts: "30"
       silent_mode_enabled: true
       splitting:
         size: "1099511627776"
       vm_snapshot_reattempts:
         enabled: true
         interval:
           type: MINUTES
           count: "5"
         max_attempts: "3"
       vss:
         enabled: true
         provider: TARGET_SYSTEM_DEFINED
       archive:
         name: '''[Machine Name]-[Plan ID]-[Unique ID]A'''
       performance_window:
         enabled: true
       retention: {}
       scheduling:
         backup_sets:
           - time:
               weekdays:
                 - FRIDAY
               repeat_at:
                 - hour: "3"
               type: WEEKLY
             type: TYPE_AUTO
         enabled: true
         max_parallel_backups: "2"
         rand_max_delay:
           type: MINUTES
           count: "30"
         scheme: ALWAYS_INCREMENTAL
         weekly_backup_day: MONDAY
       cbt: ENABLE_AND_USE
       fast_backup_enabled: true
       quiesce_snapshotting_enabled: true
     folder_id: b1g86q4m5vej********
     ```

     For more information about the `yc backup policy create` command, see the [CLI reference](../../cli/cli-ref/managed-services/backup/policy/create.md).

- API {#api}

  To create a [backup policy](../../backup/concepts/policy.md), use the [create](../../backup/backup/api-ref/Policy/create.md) REST API method for the [Policy](../../backup/backup/api-ref/Policy/index.md) resource or the [PolicyService/Create](../../backup/backup/api-ref/grpc/policy_service.md#Create) gRPC API call.

{% endlist %}

## Create a VM {#create-vm}

{% list tabs group=instructions %}

- Management console {#console}

  1. In the [management console]({{ link-console-main }}), select the folder you want to create a VM in.
  1. In the list of services, select **{{ ui-key.yacloud.iam.folder.dashboard.label_compute }}**.
  1. In the left-hand panel, select ![image](../../_assets/console-icons/server.svg) **{{ ui-key.yacloud.compute.switch_instances }}** and click **{{ ui-key.yacloud.compute.instances.button_create }}**.
  1. Enter a name for the VM: `backup-instance`.
  1. Select the `{{ region-id }}-d` [availability zone](../../overview/concepts/geo-scope.md).
  1. In the **{{ ui-key.yacloud.compute.instances.create.section_image }}** section on the **{{ ui-key.yacloud.compute.instances.create.image_value_os-products }}** tab, select `Ubuntu 22.04`.
  1. Under **{{ ui-key.yacloud.compute.instances.create.section_network }}**:

     1. Select the `cloud-network-{{ region-id }}-d` subnet.
     1. In the **{{ ui-key.yacloud.component.compute.network-select.field_external }}** field, select `{{ ui-key.yacloud.component.compute.network-select.switch_auto }}`.
     1. Select the `backup-sg` security group.

  1. Under **{{ ui-key.yacloud.compute.instances.create.section_access }}**:

     1. Select the `backup-sa` service account.
     1. Enter the `vm-user` username into the **{{ ui-key.yacloud.compute.instances.create.field_user }}** field.
     1. In the **{{ ui-key.yacloud.compute.instances.create.field_key }}** field, paste the contents of the [public key](../../compute/operations/vm-connect/ssh.md#creating-ssh-keys) file. You need to [create](../../compute/operations/vm-connect/ssh.md#creating-ssh-keys) a key pair for the SSH connection yourself.

  1. Under **{{ ui-key.yacloud.compute.instances.create.section_additional }}**, enable the **{{ ui-key.yacloud.compute.instances.create.section_backup }}** option.
  1. Under **{{ ui-key.yacloud.common.metadata }}**, add a field with the `cloudbackup` key and the following value: `{"initialPolicies": ["<daily_backup_policy_ID>"]}`.
  1. Click **{{ ui-key.yacloud.compute.instances.create.button_create }}**.

- {{ yandex-cloud }} CLI {#cli}
  
  1. Describe the custom metadata configuration in the `user-data.yaml` file:

     ```yaml
     #cloud-config
     datasource:
      Ec2:
       strict_id: false
     ssh_pwauth: no
     users:
     - name: vm-user
       sudo: ALL=(ALL) NOPASSWD:ALL
       shell: /bin/bash
       ssh_authorized_keys:
       - <public_SSH_key>
     packages:
       - curl
       - perl
       - jq
     runcmd:
       - curl https://storage.yandexcloud.net/backup-distributions/agent_installer.sh | sudo bash
     ```

  1. Specify the `daily-backup` policy ID in the `cloudbackup.json` file:

     ```json
     {"initialPolicies": ["<daily_backup_policy_ID>"]}
     ```

  1. Run this command:

     ```bash
     yc compute instance create \
       --name backup-instance \
       --zone {{ region-id }}-d \
       --network-interface subnet-name=cloud-network-{{ region-id }}-d,security-group-ids=<backup-sg_security_group_ID>,ipv4-address=auto,nat-ip-version=ipv4 \
       --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2204-lts,size=15 \
       --metadata-from-file user-data=./user-data.yaml,cloudbackup=./cloudbackup.json \
       --service-account-name backup-sa
     ```

     For more information about the `yc compute instance create` command, see the [CLI reference](../../cli/cli-ref/managed-services/compute/instance/create.md).

- API {#api}

  To create a VM, use the [create](../../compute/api-ref/Instance/create.md) REST API method for the [Instance](../../compute/api-ref/Instance/index.md) resource or the [InstanceService/Create](../../compute/api-ref/grpc/instance_service.md#Create) gRPC API call.
  
  In the request body, specify:

  * In the `metadata` field, the `user-data` object containing the custom metadata configuration with a script to install a backup agent.
  * In the `cloudbackup` field, the backup policy ID.
  
  Use `\n` as a line separator.

  {% cut "Request body example" %}
  
  ```json
  {
    "folderId": "folder_ID",
    "name": "backup-instance",
    "zoneId": "ru-central1-d",
    "platformId": "standard-v3",
    "resourcesSpec": {
      "memory": "2147483648",
      "cores": "2"
    },
    "metadata": {
      "user-data": "#cloud-config\ndatasource:\nEc2:\n  strict_id: false\nssh_pwauth: no\nusers:\n- name: vm-user\n  shell: /bin/bash\n  sudo: ALL=(ALL) NOPASSWD:ALL\n  ssh-authorized-keys:\n  - <public_SSH_key>\npackages:\n  - curl\n  - perl\n  - jq\nruncmd:\n  - curl https://storage.yandexcloud.net/backup-distributions/agent_installer.sh | sudo bash",
      "cloudbackup": "{\"initialPolicies\": [\"policy_ID\"]}"
    },
    "bootDiskSpec": {
      "diskSpec": {
        "size": "16106127360",
        "imageId": "fd8ljvsrm3l1q2tgqji9"
      }
    },
    "networkInterfaceSpecs": [
      {
        "subnetId": "subnet_ID",
        "primaryV4AddressSpec": {
          "oneToOneNatSpec": {
            "ipVersion": "IPV4"
          }
        },
        "securityGroupIds": [
          "security_group_ID"
        ]
      }
    ],
    "serviceAccountId": "service_account_ID"
  }
  ```

  {% endcut %}
  
{% endlist %}

{% note info %}

{% include [agent-installation-timespan](../../_includes/backup/agent-installation-timespan.md) %}

Policy is linked asynchronously after you create and initialize a VM, as well as install and configure a backup agent. This may take up to 10-15 minutes. As a result, the virtual machine will appear in the list of {{ backup-name }} VMs and in the list of VMs linked to the `daily-backup` policy.

{% endnote %}

You can monitor the installation progress using the VM [serial port](../../compute/operations/vm-info/get-serial-port-output.md) in the management console.

## How to delete the resources you created {#clear-out}

To stop paying for the resources you created:
1. [Delete](../../backup/operations/delete-vm.md) the VM from {{ backup-name }}.
1. [Delete](../../compute/operations/vm-control/vm-delete.md) the VM from {{ compute-name }}.
1. [Delete](../../backup/operations/backup-vm/delete.md) VM backups, if any.
