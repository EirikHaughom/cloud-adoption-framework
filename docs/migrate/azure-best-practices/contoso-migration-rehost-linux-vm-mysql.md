---
title: Rehost an on-premises Linux application to Azure VMs and Azure Database for MySQL
description: Learn how Contoso rehosts an on-premises Linux app by migrating it to Azure VMs and Azure Database for MySQL.
author: givenscj
ms.author: abuck
ms.date: 07/01/2020
ms.topic: conceptual
ms.service: cloud-adoption-framework
ms.subservice: migrate
services: azure-migrate
---

<!-- cSpell:ignore givenscj OSTICKETWEB OSTICKETMYSQL contosohost vcenter contosodc contosoosticket osticket InnoDB binlog systemctl NSGs -->

# Rehost an on-premises Linux application to Azure VMs and Azure Database for MySQL

This article shows how the fictional company Contoso rehosts a two-tier [LAMP-based](https://wikipedia.org/wiki/LAMP_(software_bundle)) application, migrating it from on-premises to Azure using Azure VMs and Azure Database for MySQL.

osTicket, the service desk application used in this example, is provided as open source. If you'd like to use it for your own testing, you can download it from [GitHub](https://github.com/osTicket/osTicket).

## Business drivers

The IT leadership team has worked closely with business partners to understand what they want to achieve:

- **Address business growth.** Contoso is growing, and as a result there's pressure on the on-premises systems and infrastructure.
- **Limit risk.** The service desk application is critical for the business. Contoso wants to move it to Azure with zero risk.
- **Extend.** Contoso doesn't want to change the application right now. It simply wants to keep the application stable.

## Migration goals

The Contoso cloud team has pinned down goals for this migration, in order to determine the best migration method:

- After migration, the application in Azure should have the same performance capabilities as it does today in their on-premises VMware environment. The application will remain as critical in the cloud as it is on-premises.
- Contoso doesn't want to invest in this application. It's important to the business, but in its current form Contoso simply want to move it safely to the cloud.
- Having completed a couple of Windows application migrations, Contoso wants to learn how to use a Linux-based infrastructure in Azure.
- Contoso wants to minimize database admin tasks after the application is moved to the cloud.

## Proposed architecture

In this scenario:

- Currently the application is tiered across two VMs (`OSTICKETWEB` and `OSTICKETMYSQL`).
- The VMs are located on VMware ESXi host `contosohost1.contoso.com` (version 6.5).
- The VMware environment is managed by vCenter Server 6.5 (`vcenter.contoso.com`), running on a VM.
- Contoso has an on-premises datacenter (`contoso-datacenter`), with an on-premises domain controller (`contosodc1`).
- The web application on `OSTICKETWEB` will be migrated to an Azure IaaS VM.
- The application database will be migrated to the Azure Database for MySQL PaaS service.
- Since Contoso is migrating a production workload, the resources will reside in the production resource group `ContosoRG`.
- The `OSTICKETWEB` resource will be replicated to the primary region (East US 2), and placed in the production network (`VNET-PROD-EUS2`):
  - The web VM will reside in the front-end subnet (`PROD-FE-EUS2`).
- The application database will be migrated to Azure Database for MySQL using the [Azure Database Migration Service](https://docs.microsoft.com/azure/dms/dms-overview).
- The on-premises VMs in the Contoso datacenter will be decommissioned after the migration is done.

    ![Scenario architecture](./media/contoso-migration-rehost-linux-vm-mysql/architecture.png)

## Migration process

Contoso will complete the migration process as follows:

To migrate the web VM:

- As a first step, Contoso sets up the Azure and on-premises infrastructure needed to deploy Azure Migrate.
- They already have the [Azure infrastructure](./contoso-migration-infrastructure.md) in place, so Contoso just needs to add and configure the replication of the VMs through the Azure Migrate: Server Migration tool.
- With everything prepared, Contoso can start replicating the VM.
- After replication is enabled and working, Contoso will complete the move using Azure Migrate.

To migrate the database:

1. Contoso provisions a MySQL instance in Azure.
2. Contoso sets up Azure Database Migration Service, ensuring access to the on-premises database server.
3. Contoso migrates the database to Azure Database for MySQL.

    ![Migration process](./media/contoso-migration-rehost-linux-vm-mysql/migration-process.png)

### Azure services

| Service | Description | Cost |
| --- | --- | --- |
| [Azure Migrate](https://docs.microsoft.com/azure/migrate/migrate-services-overview) | Contoso uses the Azure Migrate service to assess its VMware VMs. Azure Migrate assesses the migration suitability of the machines. It provides sizing and cost estimates for running in Azure. | [Azure Migrate](https://azure.microsoft.com/pricing/details/azure-migrate) is available at no additional charge, however, you may incur charges depending on the tools (first-party or ISV) you decide to use for assessment and migration. |
| [Azure Database Migration Service](https://docs.microsoft.com/azure/dms/dms-overview) | Azure Database Migration Service enables seamless migration from multiple database sources to Azure data platforms with minimal downtime. | Learn about [supported regions](https://docs.microsoft.com/azure/dms/dms-overview#regional-availability) and [Azure Database Migration Service pricing](https://azure.microsoft.com/pricing/details/database-migration). |
| [Azure Database for MySQL](https://docs.microsoft.com/azure/mysql) | The database is based on the open-source MySQL database engine. It provides a fully managed enterprise-ready community MySQL database for application development and deployment. | Learn more about Azure Database for MySQL [pricing](https://azure.microsoft.com/pricing/details/mysql) and scalability options. |

## Prerequisites

Here's what Contoso needs for this scenario.

| Requirements | Details |
| --- | --- |
| **Azure subscription** | Contoso created subscriptions during an earlier article. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free). <br><br> If you create a free account, you're the administrator of your subscription and can perform all actions. <br><br> If you use an existing subscription and you're not the administrator, you need to work with the admin to assign you Owner or Contributor permissions. <br><br> If you need more granular permissions, review [this article](https://docs.microsoft.com/azure/site-recovery/site-recovery-role-based-linked-access-control). |
| **Azure infrastructure** | Contoso set up the Azure infrastructure as described in [Azure infrastructure for migration](./contoso-migration-infrastructure.md). |
| **On-premises servers** | The on-premises vCenter Server should be running version 5.5, 6.0, 6.5 or 6.7. <br><br> An ESXi host running version 5.5, 6.0, 6.5 or 6.7. <br><br> One or more VMware VMs running on the ESXi host. |
| **On-premises VMs** | [Review Linux machines](https://docs.microsoft.com/azure/virtual-machines/linux/endorsed-distros) that are endorsed to run on Azure. |

## Scenario steps

Here's how Contoso admins will complete the migration:

> [!div class="checklist"]
>
> - **Step 1: Prepare Azure for Azure Migrate: Server Migration.** They add the server migration tool to their Azure Migrate project.
> - **Step 2: Prepare on-premises VMware for Azure Migrate: Server Migration.** They prepare accounts for VM discovery, and prepare to connect to Azure VM after migrated.
> - **Step 3: Replicate VMs.** They set up replication, and start replicating VMs to Azure Storage.
> - **Step 4: Migrate the application VM with Azure Migrate: Server Migration.** They run a test migration to make sure everything's working, and then run a full migration to move the VM to Azure.
> - **Step 5: Migrate the database.** They set up migration using Azure Database Migration Service.

## Step 1: Prepare Azure for the Azure Migrate: Server Migration tool

Here are the Azure components Contoso needs to migrate the VMs to Azure:

- A VNet in which Azure VMs will be located when they're created during migration.
- The Azure Migrate: Server Migration tool (OVA) provisioned and configured.

They set these up as follows:

1. Set up a network-Contoso already set up a network that can be for Azure Migrate: Server Migration when they [deployed the Azure infrastructure](./contoso-migration-infrastructure.md)

2. Provision the Azure Migrate: Server Migration tool.

    - From Azure Migrate, download the OVA image and import it into VMware.

        ![Download the OVA file](./media/contoso-migration-rehost-vm/migration-download-ova.png)

    - Start the imported image and configure the tool, including the following steps:

      - Set up the prerequisites.

        ![Configure the tool](./media/contoso-migration-rehost-vm/migration-setup-prerequisites.png)

      - Point the tool to the Azure subscription.

        ![Configure the tool](./media/contoso-migration-rehost-vm/migration-register-azure.png)

      - Set the VMware vCenter credentials.

        ![Configure the tool](./media/contoso-migration-rehost-vm/migration-vcenter-server.png)

      - Add any Linux-based credentials for discovery.

        ![Configure the tool](./media/contoso-migration-rehost-vm/migration-credentials.png)

3. Once configured, it will take some time for the tool to enumerate all the virtual machines. Once complete, you will see them populate in the Azure Migrate tool in Azure.

**Need more help?**

Learn about setting up the [Azure Migrate: Server Migration tool](https://docs.microsoft.com/azure/migrate/migrate-services-overview#azure-migrate-server-migration-tool).

## Step 2: Prepare on-premises VMware for Azure Migrate: Server Migration

After migrating to Azure, Contoso wants to be able to connect to the replicated VMs in Azure. To do this, there's a couple of things that the Contoso admins need to do:

- To access Azure VM, they enable SSH on the on-premises Linux VM before migration. For Ubuntu this can be completed using the following command: `sudo apt-get ssh install -y`.

- After they run the migration they can check **boot diagnostics** to view a screenshot of the VM.

- If this doesn't work, they'll need to check that the VM is running, and review these [troubleshooting tips](https://social.technet.microsoft.com/wiki/contents/articles/31666.troubleshooting-remote-desktop-connection-after-failover-using-asr.aspx).

- Install the [Azure Linux agent](https://docs.microsoft.com/azure/virtual-machines/extensions/agent-linux).

**Need more help?**

- Learn about [preparing VMs for migration](https://docs.microsoft.com/azure/migrate/prepare-for-migration).

## Step 3: Replicate VM

Before Contoso admins can run a migration to Azure, they need to set up and enable replication.

With discovery completed, they can begin replication of the application VM to Azure.

1. In the Azure Migrate project > **Servers** > **Azure Migrate: Server Migration**, select **Replicate**.

    ![Replicate VMs](./media/contoso-migration-rehost-linux-vm/select-replicate.png)

2. In **Replicate** > **Source settings** > **Are your machines virtualized?**, select **Yes, with VMware vSphere**.

3. In **On-premises appliance**, select the name of the Azure Migrate appliance that you set up > **OK**.

    ![Source settings](./media/contoso-migration-rehost-linux-vm/source-settings.png)

4. In **Virtual machines**, select the machines you want to replicate.
    - If you've run an assessment for the VMs, you can apply VM sizing and disk type (premium/standard) recommendations from the assessment results. To do this, in **Import migration settings from an Azure Migrate assessment?**, select the **Yes** option.
    - If you didn't run an assessment, or you don't want to use the assessment settings, select the **No** option.
    - If you selected to use the assessment, select the VM group, and assessment name.

    ![Select assessment](./media/contoso-migration-rehost-linux-vm/select-assessment.png)

5. In **Virtual machines**, search for VMs as needed, and check each VM you want to migrate. Then select **Next: Target settings**.

6. In **Target settings**, select the subscription, and target region to which you'll migrate, and specify the resource group in which the Azure VMs will reside after migration. In **Virtual Network**, select the Azure VNet/subnet to which the Azure VMs will be joined after migration.

7. In **Azure Hybrid Benefit**, select the following:

    - Select **No** if you don't want to apply Azure Hybrid Benefit. Then select **Next**.

8. In **Compute**, review the VM name, size, OS disk type, and availability set. VMs must conform with [Azure requirements](https://docs.microsoft.com/azure/migrate/migrate-support-matrix-vmware#vmware-requirements).

    - **VM size:** If you're using assessment recommendations, the VM size dropdown will contain the recommended size. Otherwise Azure Migrate picks a size based on the closest match in the Azure subscription. Alternatively, pick a manual size in **Azure VM size.**
    - **OS disk:** Specify the OS (boot) disk for the VM. The OS disk is the disk that has the operating system bootloader and installer.
    - **Availability set:** If the VM should be in an Azure availability set after migration, specify the set. The set must be in the target resource group you specify for the migration.

9. In **Disks**, specify whether the VM disks should be replicated to Azure, and select the disk type (standard SSD/HDD or premium-managed disks) in Azure. Then select **Next**.
    - You can exclude disks from replication.
    - If you exclude disks, won't be present on the Azure VM after migration.

10. In **Review and start replication**, review the settings, then select **Replicate** to start the initial replication for the servers.

> [!NOTE]
> You can update replication settings any time before replication starts, in **Manage** > **Replicating machines**. Settings can't be changed after replication starts.

## Step 4: Migrate the VM with Azure Migrate: Server Migration

Contoso admins run a quick test migration, and then a full migration to move the web VM.

### Run a test migration

1. In **Migration goals** > **Servers** > **Azure Migrate: Server Migration**, select **Test migrated servers**.

     ![Test migrated servers](./media/contoso-migration-rehost-linux-vm/test-migrated-servers.png)

2. Select and hold (or right-click) the VM to test, then select **Test migrate**.

    ![Test migration](./media/contoso-migration-rehost-linux-vm/test-migrate.png)

3. In **Test Migration**, select the Azure VNet in which the Azure VM will be located after the migration. We recommend you use a nonproduction VNet.
4. The **Test migration** job starts. Monitor the job in the portal notifications.
5. After the migration finishes, view the migrated Azure VM in **Virtual Machines** in the Azure portal. The machine name has a suffix **-Test**.
6. After the test is done, select and hold (or right-click) the Azure VM in **Replicating machines**, then select **Clean up test migration**.

    ![Clean up migration](./media/contoso-migration-rehost-linux-vm/clean-up.png)

### Migrate the VM

Now Contoso admins run a full migration to complete the move.

1. In the Azure Migrate project > **Servers** > **Azure Migrate: Server Migration**, select **Replicating servers**.

    ![Replicating servers](./media/contoso-migration-rehost-linux-vm/replicating-servers.png)

2. In **Replicating machines**, select and hold (or right-click) the VM > **Migrate**.
3. In **Migrate** > **Shut down virtual machines and perform a planned migration with no data loss**, select **Yes** > **OK**.
    - By default Azure Migrate shuts down the on-premises VM, and runs an on-demand replication to synchronize any VM changes that occurred since the last replication occurred. This ensures no data loss.
    - If you don't want to shut down the VM, select **No**.
4. A migration job starts for the VM. Track the job in Azure notifications.
5. After the job finishes, you can view and manage the VM from the **Virtual Machines** page.

## Step 5: Provision Azure Database for MySQL

Contoso admins provision a MySQL database instance in the primary region (`East US 2`).

1. In the Azure portal, they create an Azure Database for MySQL resource.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/mysql-1.png)

2. They add the name `contosoosticket` for the Azure database. They add the database to the production resource group `ContosoRG`, and specify credentials for it.
3. The on-premises MySQL database is version 5.7, so they select this version for compatibility. They use the default sizes, which match their database requirements.

     ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/mysql-2.png)

4. For **Backup Redundancy Options**, they select to use **Geo-Redundant**. This option allows them to restore the database in their secondary region (`Central US`) if an outage occurs. They can only configure this option when they provision the database.

     ![Redundancy](./media/contoso-migration-rehost-linux-vm-mysql/db-redundancy.png)

5. In the `VNET-PROD-EUS2` network > **Service endpoints**, they add a service endpoint (a database subnet) for the SQL service.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/mysql-3.png)

6. After adding the subnet, they create a virtual network rule that allows access from the database subnet in the production network.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/mysql-4.png)

## Step 6: Migrate the database

There are several ways to move the MySQL database. Each option requires you to create an Azure Database for MySQL instance for the target. Once created, you can perform the migration using two paths:

- 6A: Azure Database Migration Service
- 6b: MySQL Workbench backup and restore

### Step 6a: Migrate the database via Azure Database Migration Service

Contoso admins migrate the database via Azure Database Migration Service by following the [step-by-step migration tutorial](https://docs.microsoft.com/azure/dms/tutorial-mysql-azure-mysql-online). They can perform both online, offline and hybrid (preview) migrations using MySQL 5.6 or 5.7.

> [!NOTE]
> MySQL 8.0 is supported in Azure Database for MySQL, but the Database Migration Service tool does not yet support that version.

As a summary, you must perform the following:

- Ensure all migration prerequisites are met:

  - The MySQL server database source must match the version that Azure Database for MySQL supports. Azure Database for MySQL supports MySQL Community Edition, the InnoDB storage engine, and migration across source and target with same versions.
  - Enable binary logging in `my.ini` (Windows) or `my.cnf` (Unix). Failure to do this will cause the following error in the Migration Wizard: `Error in binary logging. Variable binlog_row_image has value 'minimal'. Please change it to 'full. For more information, see https://go.microsoft.com/fwlink/?linkid=873009`.
  - User must have `ReplicationAdmin` role.
  - Migrate the database schemas without foreign keys and triggers.

- Create a virtual network that connects via ExpressRoute or VPN to your on-premises network.

- Create an Azure Database Migration Service instance using a `Premium` SKU that is connected to the VNet.

- Ensure that the instance can access the MySQL database via the virtual network. This would entail ensuring that all incoming ports are allowed from Azure to MySQL at the virtual network level, the network VPN, and the machine that hosts MySQL.

- Run the Database Migration Service tool:

  - Create a migration project.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-new-project.png)

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-new-project-02.png)

  - Add a source (on-premises database).

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-source.png)

  - Select a target.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-target.png)

  - Select the databases to migrate.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-databases.png)

  - Configure advanced settings.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-settings.png)

  - Start the replication and resolve any errors.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-monitor.png)

  - Perform the final cutover. ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-cutover.png)

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-cutover-complete.png)

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-cutover-complete-02.png)

  - Reinstate any foreign keys and triggers.

  - Modify applications to use the new database.

    ![MySQL](./media/contoso-migration-rehost-linux-vm-mysql/migration-dms-cutover-apps.png)

### Step 6b: Migrate the database (MySQL Workbench)

Contoso admins migrate the database using backup and restore, with MySQL tools. They install MySQL Workbench, back up the database from `OSTICKETMYSQL`, and then restore it to Azure Database for MySQL.

### Install MySQL Workbench

1. They check the [prerequisites and downloads MySQL Workbench](https://dev.mysql.com/downloads/workbench/?utm_source=tuicool).

2. They install MySQL Workbench for Windows in accordance with the [installation instructions](https://dev.mysql.com/doc/workbench/en/wb-installing.html).

3. In MySQL Workbench, they create a MySQL connection to OSTICKETMYSQL.

    ![MySQL Workbench](./media/contoso-migration-rehost-linux-vm-mysql/workbench1.png)

4. They export the database as `osticket` to a local self-contained file.

    ![MySQL Workbench](./media/contoso-migration-rehost-linux-vm-mysql/workbench2.png)

5. After the database has been backed up locally, they create a connection to the Azure Database for MySQL instance.

    ![MySQL Workbench](./media/contoso-migration-rehost-linux-vm-mysql/workbench3.png)

6. Now, they can import (restore) the database in the Azure Database for MySQL instance, from the self-contained file. A new schema (`osticket`) is created for the instance.

    ![MySQL Workbench](./media/contoso-migration-rehost-linux-vm-mysql/workbench4.png)

### Connect the VM to the database

As the final step in the migration process, Contoso admins update the connection string of the application to point to the application database running on the `OSTICKETMYSQL` VM.

1. They make an SSH connection to the `OSTICKETWEB` VM using PuTTY or another SSH client. The VM is private so they connect using the private IP address.

    ![Connect to database](./media/contoso-migration-rehost-linux-vm/db-connect.png)

    ![Connect to database](./media/contoso-migration-rehost-linux-vm/db-connect2.png)

2. They need to make sure that the `OSTICKETWEB` VM can communicate with the `OSTICKETMYSQL` VM. Currently the configuration is hardcoded with the on-premises IP address `172.16.0.43`.

    **Before the update:**

    ![Update IP](./media/contoso-migration-rehost-linux-vm/update-ip1.png)

    **After the update:**

    ![Update IP](./media/contoso-migration-rehost-linux-vm/update-ip2.png)

3. They restart the service with `systemctl restart apache2`.

    ![Restart](./media/contoso-migration-rehost-linux-vm/restart.png)

4. Finally, they update the DNS records for `OSTICKETWEB` and `OSTICKETMYSQL`, on one of the Contoso domain controllers.

    ![Update DNS](./media/contoso-migration-rehost-linux-vm-mysql/update-dns.png)

    ![Update DNS](./media/contoso-migration-rehost-linux-vm-mysql/update-dns.png)

**Need more help?**

- Learn about [running a test migration](https://docs.microsoft.com/azure/migrate/tutorial-migrate-vmware#run-a-test-migration).
- Learn about [migrating VMs to Azure](https://docs.microsoft.com/azure/migrate/tutorial-migrate-vmware#migrate-vms).

## Review the deployment

With the application now running, Contoso need to fully operationalize and secure their new infrastructure.

## Clean up after migration

With migration complete, the osTicket application tiers are running on Azure VMs.

Now, Contoso needs to do the following:

- Remove the VMware VMs from the vCenter inventory.
- Remove the on-premises VMs from local backup jobs.
- Update internal documentation show new locations and IP addresses.
- Review any resources that interact with the on-premises VMs, and update any relevant settings or documentation to reflect the new configuration.
- Contoso used the Azure Migrate service with dependency mapping to assess the `OSTICKETWEB` VM for migration.

### Security

The Contoso security team review the VM and database to determine any security issues.

- They review the network security groups (NSGs) for the VM, to control access. NSGs are used to ensure that only traffic allowed to the application can pass.
- They consider securing the data on the VM disks using Azure Disk Encryption and Azure Key Vault.
- Communication between the VM and database instance isn't configured for SSL. They will need to do this to ensure that database traffic can't be hacked.

For more information, see [Security best practices for IaaS workloads in Azure](https://docs.microsoft.com/azure/security/fundamentals/iaas).

<!-- docsTest:ignore "Quickstart: Set" -->

### BCDR

For business continuity and disaster recovery, Contoso takes the following actions:

- **Keep data safe.** Contoso backs up the data on the application VM using [Azure VM backup](https://docs.microsoft.com/azure/backup/backup-azure-vms-introduction). They don't need to configure backup for the database. Azure Database for MySQL automatically creates and stores server backups. They selected to use geo-redundancy for the database, so it's resilient and production-ready.

- **Keep applications up and running.** Contoso replicates the application VMs in Azure to a secondary region using Site Recovery. For more information, see [Quickstart: Set up disaster recovery to a secondary Azure region for an Azure VM](https://docs.microsoft.com/azure/site-recovery/azure-to-azure-quickstart).

### Licensing and cost optimization

- After deploying resources, Contoso assigns Azure tags, in accordance with decisions they made during the [Azure infrastructure](./contoso-migration-infrastructure.md#set-up-tagging) deployment.
- There are no licensing issues for the Contoso Ubuntu servers.
- Contoso will use [Azure Cost Management and Billing](https://docs.microsoft.com/azure/cost-management-billing/cost-management-billing-overview) to ensure they stay within budgets established by their IT leadership.
