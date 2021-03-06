---
title: Azure Site Recovery troubleshooting for Azure-to-Azure replication errors| Microsoft Docs
description: Troubleshoot errors when replicating Azure virtual machines for disaster recovery.
services: site-recovery
author: asgang
manager: rochakm
ms.service: site-recovery
ms.topic: article
ms.date: 04/08/2019
ms.author: asgang

---
# Troubleshoot Azure-to-Azure VM replication errors

This article describes how to troubleshoot common errors in Azure Site Recovery during replication and recovery of Azure virtual machines (VMs) from one region to another. For more information about supported configurations, see the [support matrix for replicating Azure VMs](site-recovery-support-matrix-azure-to-azure.md).

## <a name="azure-resource-quota-issues-error-code-150097"></a>Azure resource quota issues (error code 150097)

Make sure your subscription is enabled to create Azure VMs in the target region that you plan to use as your disaster-recovery region. Also make sure your subscription has sufficient quota to create VMs of the necessary sizes. By default, Site Recovery chooses a target VM size that's the same as the source VM size. If the matching size isn't available, Site Recovery automatically chooses the closest available size.

If there's no size that supports the source VM configuration, the following message appears:

> "Replication couldn't be enabled for the virtual machine *VmName*."

### Possible causes

- Your subscription ID isn't enabled to create any VMs in the target region location.
- Your subscription ID isn't enabled, or doesn't have sufficient quota, to create specific VM sizes in the target region location.
- No suitable target VM size is found to match the source VM's network interface card (NIC) count (2), for the subscription ID in the target region location.

### Fix the problem

Contact [Azure billing support](https://docs.microsoft.com/azure/azure-supportability/resource-manager-core-quotas-request) to enable your subscription to create VMs of the required sizes in the target location. Then, retry the failed operation.

If the target location has a capacity constraint, disable replication to it. Then, enable replication to a different location where your subscription has sufficient quota to create VMs of the required sizes.

## <a name="trusted-root-certificates-error-code-151066"></a>Trusted root certificates (error code 151066)

If not all the latest trusted root certificates are present on the VM, your "Enable replication" Site Recovery job might fail. Authentication and authorization of Site Recovery service calls from the VM fail without these certificates. 

If the "Enable replication" job fails, the following message appears:

> "Site Recovery configuration failed."

### Possible cause

The trusted root certificates required for authorization and authentication aren't present on the virtual machine.

### Fix the problem

#### Windows

For a VM running the Windows operating system, install the latest Windows updates on the VM so that all the trusted root certificates are present on the machine. Follow the typical Windows update-management or certificate update-management process in your organization to get the latest root certificates and the updated certificate-revocation list on the VMs.

If you're in a disconnected environment, follow the standard Windows update process in your organization to get the certificates. If the required certificates aren't present on the VM, the calls to the Site Recovery service fail for security reasons.

To verify that the issue is resolved, go to login.microsoftonline.com from a browser in your VM.

For more information, see [Configure trusted roots and disallowed certificates](https://technet.microsoft.com/library/dn265983.aspx).

#### Linux

Follow the guidance provided by the distributor of your Linux operating system version to get the latest trusted root certificates and the latest certificate-revocation list on the VM.

Because SuSE Linux uses symbolic links (or *symlinks*) to maintain a certificate list, follow these steps:

1. Sign in as a root user.

1. Run this command to change the directory:

    **# cd /etc/ssl/certs**

1. Check whether the Symantec root CA certificate is present:

    **# ls VeriSign_Class_3_Public_Primary_Certification_Authority_G5.pem**

1. If the Symantec root CA certificate is not found, run the following command to download the file. Check for any errors and follow recommended actions for network failures.

    **# wget https://www.symantec.com/content/dam/symantec/docs/other-resources/verisign-class-3-public-primary-certification-authority-g5-en.pem -O VeriSign_Class_3_Public_Primary_Certification_Authority_G5.pem**

1. Check whether the Baltimore root CA certificate is present:

    **# ls Baltimore_CyberTrust_Root.pem**

1. If the Baltimore root CA certificate is not found, run this command to download the certificate:

    **# wget https://www.digicert.com/CACerts/BaltimoreCyberTrustRoot.crt.pem -O Baltimore_CyberTrust_Root.pem**

1. Check whether the DigiCert_Global_Root_CA certificate is present:

    **# ls DigiCert_Global_Root_CA.pem**

1. If the DigiCert_Global_Root_CA is not found, run the following commands to download the certificate:

    **# wget http://www.digicert.com/CACerts/DigiCertGlobalRootCA.crt**

    **# openssl x509 -in DigiCertGlobalRootCA.crt -inform der -outform pem -out DigiCert_Global_Root_CA.pem**

1. Run the rehash script to update the certificate subject hashes for the newly downloaded certificates:

    **# c_rehash**

1. Run these commands to check whether the subject hashes as symlinks have been created for the certificates:

    - Command:

        **# ls -l | grep Baltimore**

    - Output:

        `lrwxrwxrwx 1 root root   29 Jan  8 09:48 3ad48a91.0 -> Baltimore_CyberTrust_Root.pem`

        `-rw-r--r-- 1 root root 1303 Jun  5  2014 Baltimore_CyberTrust_Root.pem`

    - Command:

        **# ls -l | grep VeriSign_Class_3_Public_Primary_Certification_Authority_G5**

    - Output:

        `-rw-r--r-- 1 root root 1774 Jun  5  2014 VeriSign_Class_3_Public_Primary_Certification_Authority_G5.pem`

        `lrwxrwxrwx 1 root root   62 Jan  8 09:48 facacbc6.0 -> VeriSign_Class_3_Public_Primary_Certification_Authority_G5.pem`

    - Command:

        **# ls -l | grep DigiCert_Global_Root**

    - Output:

        `lrwxrwxrwx 1 root root   27 Jan  8 09:48 399e7759.0 -> DigiCert_Global_Root_CA.pem`

        `-rw-r--r-- 1 root root 1380 Jun  5  2014 DigiCert_Global_Root_CA.pem`

1. Create a copy of the file VeriSign_Class_3_Public_Primary_Certification_Authority_G5.pem with filename b204d74a.0:

    **# cp VeriSign_Class_3_Public_Primary_Certification_Authority_G5.pem b204d74a.0**

1. Create a copy of the file Baltimore_CyberTrust_Root.pem with filename 653b494a.0:

    **# cp Baltimore_CyberTrust_Root.pem 653b494a.0**

1. Create a copy of the file DigiCert_Global_Root_CA.pem with filename 3513523f.0:

    **# cp DigiCert_Global_Root_CA.pem 3513523f.0**

1. Check whether the files are present:

    - Command:

        **# ls -l 653b494a.0 b204d74a.0 3513523f.0**

    - Output

        `-rw-r--r-- 1 root root 1774 Jan  8 09:52 3513523f.0`

        `-rw-r--r-- 1 root root 1303 Jan  8 09:52 653b494a.0`

        `-rw-r--r-- 1 root root 1774 Jan  8 09:52 b204d74a.0`

## Outbound connectivity for Site Recovery URLs or IP ranges (error code 151037 or 151072)

For Site Recovery replication to work, outbound connectivity is required from the VM to specific URLs or IP ranges. If your VM is behind a firewall or uses network security group (NSG) rules to control outbound connectivity, you might face one of these issues.

### <a name="issue-1-failed-to-register-azure-virtual-machine-with-site-recovery-151195-br"></a>Issue 1: Failed to register the Azure virtual machine with Site Recovery (error code 151195)

#### Possible cause 

The connection to Site Recovery endpoints can't be established because of a DNS resolution failure.

This problem happens most frequently during reprotection, when you have failed over the virtual machine but the DNS server is not reachable from the disaster-recovery (DR) region.

#### Fix the problem

If you're using a custom DNS, make sure that the DNS server is accessible from the disaster-recovery region. To find out whether you have a custom DNS, on the VM, go to *disaster recovery network* > **DNS servers**.

![Custom DNS server list](./media/azure-to-azure-troubleshoot-errors/custom_dns.PNG)

Try accessing the DNS server from the virtual machine. If the server is not accessible, make it accessible either by failing over the DNS server or by creating the line of site between the DR network and the DNS.

### Issue 2: Site Recovery configuration failed (error code 151196)

#### Possible cause

The connection to Office 365 authentication and identity IP4 endpoints can't be established.

#### Fix the problem

Site Recovery requires access to Office 365 IP ranges for authentication.
If you're using Azure NSG rules or firewall proxy to control outbound network connectivity on the VM, be sure you allow communication to Office 365 IP ranges. Create an NSG rule based on an [Azure Active Directory (Azure AD) service tag](../virtual-network/security-overview.md#service-tags), allowing access to all IP addresses corresponding to Azure AD. If new addresses are added to Azure AD in the future, you must create new NSG rules.

> [!NOTE]
> If VMs are behind a *Standard* internal load balancer, the load balancer by default does not have access to Office 365 IP ranges (that is, login.microsoftonline.com). Either change the internal load balancer type to *Basic* or create outbound access as described in the article [Configure load balancing and outbound rules](https://aka.ms/lboutboundrulescli).

### Issue 3: Site Recovery configuration failed (error code 151197)

#### Possible cause

The connection can't be established to Site Recovery service endpoints.

#### Fix the problem

Site Recovery requires access to [Site Recovery IP ranges](https://docs.microsoft.com/azure/site-recovery/azure-to-azure-about-networking#outbound-connectivity-for-ip-address-ranges), depending on the region. Make sure that the required IP ranges are accessible from the virtual machine.

### Issue 4: Azure-to-Azure replication failed when the network traffic goes through an on-premises proxy server (error code 151072)

#### Possible cause

The custom proxy settings are invalid, and the Site Recovery Mobility Service agent did not auto-detect the proxy settings from Internet Explorer.

#### Fix the problem

The Mobility Service agent detects the proxy settings from Internet Explorer on Windows and from /etc/environment on Linux.

If you prefer to set the proxy only for the Mobility Service, you can provide the proxy details in the ProxyInfo.conf file in these locations:

- **Linux**: /usr/local/InMage/config/
- **Windows**: C:\ProgramData\Microsoft Azure Site Recovery\Config

In ProxyInfo.conf, provide the proxy settings in the following initialization-file format:

> [*proxy*]

> Address=*http://1.2.3.4*

> Port=*567*


> [!NOTE]
> The Site Recovery Mobility Service agent supports only *un-authenticated proxies*.

### More information

To specify [required URLs](azure-to-azure-about-networking.md#outbound-connectivity-for-urls) or [required IP ranges](azure-to-azure-about-networking.md#outbound-connectivity-for-ip-address-ranges), follow the guidance in [About networking in Azure to Azure replication](site-recovery-azure-to-azure-networking-guidance.md).

## Disk not found in the machine (error code 150039)

A new disk attached to the VM must be initialized. If the disk is not found, the following message appears:

> "Azure data disk *DiskName* *DiskURI* with logical unit number *LUN* *LUNValue* was not mapped to a corresponding disk being reported from within the VM that has the same LUN value.

### Possible causes

- A new data disk was attached to the VM but wasn't initialized.
- The data disk inside the VM is not correctly reporting the logical unit number (LUN) value at which the disk was attached to the VM.

### Fix the problem

Make sure that the data disks are initialized, and then retry the operation.

- **Windows**: [Attach and initialize a new disk](https://docs.microsoft.com/azure/virtual-machines/windows/attach-managed-disk-portal).

- **Linux**: [Initialize a new data disk in Linux](https://docs.microsoft.com/azure/virtual-machines/linux/add-disk).

If the problem persists, contact support.

## One or more disks are available for protection (error code 153039)

### Possible causes

- One or more disks were recently added to the virtual machine after protection.
- One or more disks were initialized after protection of the virtual machine.

### Fix the problem

To make the replication status of the VM healthy again, you can choose either to protect the disks or to dismiss the warning.

#### To protect the disks

1. Go to **Replicated Items** > *VM name* > **Disks**.
1. Select the unprotected disk, and then select **Enable replication**:

    ![Enable replication on VM disks](./media/azure-to-azure-troubleshoot-errors/add-disk.png)

#### To dismiss the warning

1. Go to **Replicated items** > *VM name*.
1. Select the warning in the **Overview** section, and then select **OK**.

    ![Dismiss new-disk warning](./media/azure-to-azure-troubleshoot-errors/dismiss-warning.png)

## Remove the virtual machine from the vault completed with information (error code 150225)

When it protects the virtual machine, Site Recovery creates some links on the source virtual machine. When you remove the protection or disable replication, Site Recovery removes these links as a part of the cleanup job. If the virtual machine has a resource lock, the cleanup job gets completed with the information. The information says that the virtual machine has been removed from the Recovery Services vault, but that some of the stale links couldn't be cleaned up on the source machine.

You can ignore this warning if you never intend to protect this virtual machine again. But if you have to protect this virtual machine later, follow the steps under "Fix the problem" to clean up the links.

> [!WARNING]
> If you don't do the cleanup:
>
> - When you enable replication by means of the Recovery Services vault, the virtual machine won't be listed.
> - If you try to protect the VM by using **Virtual machine** > **Settings** > **Disaster Recovery**, the operation will fail with the message "Replication cannot be enabled because of the existing stale resource links on the VM."

### Fix the problem

> [!NOTE]
> Site Recovery doesn't delete the source virtual machine or affect it in any way while you perform these steps.

1. Remove the lock from the VM or VM resource group. For example, in the following image, the resource lock on the VM named "MoveDemo" must be deleted:

    ![Remove lock from VM](./media/site-recovery-azure-to-azure-troubleshoot/vm-locks.png)

1. Download the script to [remove a stale Site Recovery configuration](https://github.com/AsrOneSdk/published-scripts/blob/master/Cleanup-Stale-ASR-Config-Azure-VM.ps1).
1. Run the script, which is called Cleanup-stale-asr-config-Azure-VM.ps1. Supply the subscription ID, VM Resource Group, and VM name as parameters.
1. If you're asked for Azure credentials, provide them. Then verify that the script runs without any failures.

## Replication can't be enabled because of stale resource links on the VM (error code 150226)

### Possible cause

The virtual machine has a stale configuration from previous Site Recovery protection.

A stale configuration can occur on an Azure VM if you enabled replication for the Azure VM by using Site Recovery, and then:

- You disabled replication, but the source VM had a resource lock.
- You deleted the Site Recovery vault without explicitly disabling replication on the VM.
- You deleted the resource group containing the Site Recovery vault without explicitly disabling replication on the VM.

### Fix the problem

> [!NOTE]
> Site Recovery doesn't delete the source virtual machine or affect it in any way while you perform these steps.

1. Remove the lock from the VM or VM resource group. For example, in the following image, the resource lock on the VM named "MoveDemo" must be deleted:

    ![Remove lock from VM](./media/site-recovery-azure-to-azure-troubleshoot/vm-locks.png)

1. Download the script to [remove a stale Site Recovery configuration](https://github.com/AsrOneSdk/published-scripts/blob/master/Cleanup-Stale-ASR-Config-Azure-VM.ps1).
1. Run the script, which is called Cleanup-stale-asr-config-Azure-VM.ps1. Supply the subscription ID, VM Resource Group, and VM name as parameters.
1. If you're asked for Azure credentials, provide them. Then verify that the script runs without any failures.

## Unable to see the Azure VM or resource group for the selection in the "Enable replication" job

### Cause 1: The resource group and source virtual machine are in different locations

Site Recovery currently requires the source region resource group and virtual machines to be in the same location. If they are not, you won't be able to find the virtual machine or resource group when you try to apply protection.

As a workaround, you can enable replication from the VM instead of the Recovery Services vault. Go to **Source VM** > **Properties** > **Disaster Recovery** and enable the replication.

### Cause 2: The resource group is not part of the selected subscription

You might not be able to find the resource group at the time of protection if the resource group is not part of the selected subscription. Make sure that the resource group belongs to the subscription that you're using.

### Cause 3: Stale configuration

You might not see the VM that you want to enable for replication if a stale Site Recovery configuration has been left on the Azure VM. This condition could occur if you enabled replication for the Azure VM by using Site Recovery, and then:

- You deleted the Site Recovery vault without explicitly disabling replication on the VM.
- You deleted the resource group containing the Site Recovery vault without explicitly disabling replication on the VM.
- You disabled replication, but the source VM had a resource lock.

### Fix the problem

> [!NOTE]
> Make sure to update the "AzureRM.Resources" module before using the script mentioned in this section.  Site Recovery doesn't delete the source virtual machine or affect it in any way while you perform these steps.

1. Remove the lock, if any, from the VM or VM resource group. For example, in the following image, the resource lock on the VM named "MoveDemo" must be deleted:

    ![Remove lock from VM](./media/site-recovery-azure-to-azure-troubleshoot/vm-locks.png)

1. Download the script to [remove a stale Site Recovery configuration](https://github.com/AsrOneSdk/published-scripts/blob/master/Cleanup-Stale-ASR-Config-Azure-VM.ps1).
1. Run the script, which is called Cleanup-stale-asr-config-Azure-VM.ps1. Supply the subscription ID, VM Resource Group, and VM name as parameters.
1. If you're asked for Azure credentials, provide them. Then verify that the script runs without any failures.

## Unable to select a virtual machine for protection

### Cause 1: The virtual machine has an extension installed in a failed or unresponsive state

Go to **Virtual machines** > **Settings** > **Extensions** and check for any extensions in a failed state. Uninstall any failed extension, and then try again to protect the virtual machine.

### Cause 2: The VM's provisioning state is not valid

See the troubleshooting steps in [The VM's provisioning state is not valid](#the-vms-provisioning-state-is-not-valid-error-code-150019), later in this article.

## The VM's provisioning state is not valid (error code 150019)

To enable replication on the VM, its provisioning state must be **Succeeded**. Follow these steps to check the provisioning state:

1. In the Azure portal, select the **Resource Explorer** from **All Services**.
1. Expand the **Subscriptions** list and select your subscription.
1. Expand the **ResourceGroups** list and select the resource group of the VM.
1. Expand the **Resources** list and select your VM.
1. Check the **provisioningState** field in Instance view on the right side.

### Fix the problem

- If **provisioningState** is **Failed**, contact support with details to troubleshoot.
- If **provisioningState** is **Updating**, another extension might be being deployed. Check whether there are any ongoing operations on the VM, wait for them to finish, and then retry the failed Site Recovery "Enable replication" job.

## Unable to select target VM (network selection tab is unavailable)

### Cause 1: Your VM is attached to a network that's already mapped to a target network

If the source VM is part of a virtual network, and another VM from the same virtual network is already mapped with a network in the target resource group, the network-selection drop-down list box is unavailable (appears dimmed) by default.

![Network selection list unavailable](./media/site-recovery-azure-to-azure-troubleshoot/unabletoselectnw.png)

### Cause 2: You previously protected the VM by using Site Recovery, and then you disabled the replication

Disabling replication of a VM doesn't delete the network mapping. The mapping must be deleted from the Recovery Services vault where the VM was protected. Go to *Recovery Services vault* > **Site Recovery Infrastructure** > **Network Mapping**.

![Delete network mapping](./media/site-recovery-azure-to-azure-troubleshoot/delete_nw_mapping.png)

The target network that was configured during the disaster-recovery setup can be changed after the initial setup, after the VM is protected:

![Modify network mapping](./media/site-recovery-azure-to-azure-troubleshoot/modify_nw_mapping.png)

Note that changing network mapping affects all protected VMs that use that same network mapping.

## COM+ or Volume Shadow Copy service error (error code 151025)

When this error occurs, the following message appears:

> "Site Recovery extension failed to install"

### Possible causes

- The COM+ System Application service is disabled.
- The Volume Shadow Copy service is disabled.

### Fix the problem

Set the COM+ System Application and Volume Shadow Copy services to automatic or manual startup mode.

1. Open the Services console in Windows.
1. Make sure the COM+ System Application and Volume Shadow Copy services are not set to **Disabled** as their **Startup Type**.

    ![Check startup type of COM+ System Application and Volume Shadow Copy services](./media/azure-to-azure-troubleshoot-errors/com-error.png)

## Unsupported managed-disk size (error code 150172)

When this error occurs, the following message appears:

> "Protection couldn't be enabled for the virtual machine as it has *DiskName* with size *DiskSize*)* that is lesser than the minimum supported size 1024 MB."

### Possible cause

The disk is smaller than the supported size of 1024 MB.

### Fix the problem

Make sure that the disk size is within the supported size range, and then retry the operation.

## <a name="enable-protection-failed-as-device-name-mentioned-in-the-grub-configuration-instead-of-uuid-error-code-151126"></a>Protection was not enabled because the GRUB configuration includes the device name instead of the UUID (error code 151126)

### Possible cause

The Linux GRUB configuration files (/boot/grub/menu.lst", /boot/grub/grub.cfg, /boot/grub2/grub.cfg, or /etc/default/grub) might specify the actual device names instead of UUID values for the *root* and *resume* parameters. Site Recovery requires UUIDs because device names can change. Upon restart, a VM might not come up with the same name on failover, resulting in problems.

The following examples are lines from GRUB files where device names (shown in bold) appear instead of the required UUIDs:

- File /boot/grub2/grub.cfg

  > linux   /boot/vmlinuz-3.12.49-11-default **root=/dev/sda2**  ${extra_cmdline} **resume=/dev/sda1** splash=silent quiet showopts

- File: /boot/grub/menu.lst

  > kernel /boot/vmlinuz-3.0.101-63-default **root=/dev/sda2** **resume=/dev/sda1** splash=silent crashkernel=256M-:128M showopts vga=0x314


### Fix the problem

Replace each device name with the corresponding UUID:

1. Find the UUID of the device by executing the command **blkid** ***device name***. For example:

    ```
    blkid /dev/sda1
    /dev/sda1: UUID="6f614b44-433b-431b-9ca1-4dd2f6f74f6b" TYPE="swap"
    blkid /dev/sda2
    /dev/sda2: UUID="62927e85-f7ba-40bc-9993-cc1feeb191e4" TYPE="ext3"
   ```

1. Replace the device name with its UUID, in the formats **root=UUID**=*UUID* and **resume=UUID**=*UUID*. For example, after replacement, the line from /boot/grub/menu.lst (discussed earlier) would look like this:

    > kernel /boot/vmlinuz-3.0.101-63-default **root=UUID=62927e85-f7ba-40bc-9993-cc1feeb191e4** **resume=UUID=6f614b44-433b-431b-9ca1-4dd2f6f74f6b** splash=silent crashkernel=256M-:128M showopts vga=0x314

1. Retry the protection.

## Enable protection failed because the device mentioned in the GRUB configuration doesn't exist (error code 151124)

### Possible cause

The GRUB configuration files (/boot/grub/menu.lst, /boot/grub/grub.cfg, /boot/grub2/grub.cfg, or /etc/default/grub) might contain the parameters *rd.lvm.lv* or *rd_LVM_LV*. These parameters identify the Logical Volume Manager (LVM) devices that are to be discovered at boot time. If these LVM devices don't exist, the protected system itself won't boot and will be stuck in the boot process. The same problem will also be seen with the failover VM. Here are few examples:

- File: /boot/grub2/grub.cfg on RHEL7:

    > linux16 /vmlinuz-3.10.0-957.el7.x86_64 root=/dev/mapper/rhel_mup--rhel7u6-root ro crashkernel=128M\@64M **rd.lvm.lv=rootvg/root rd.lvm.lv=rootvg/swap** rhgb quiet LANG=en_US.UTF-8

- File: /etc/default/grub on RHEL7:

    > GRUB_CMDLINE_LINUX="crashkernel=auto **rd.lvm.lv=rootvg/root rd.lvm.lv=rootvg/swap** rhgb quiet"

- File: /boot/grub/menu.lst on RHEL6:

    > kernel /vmlinuz-2.6.32-754.el6.x86_64 ro root=UUID=36dd8b45-e90d-40d6-81ac-ad0d0725d69e rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD SYSFONT=latarcyrheb-sun16 crashkernel=auto **rd_LVM_LV=rootvg/lv_root**  KEYBOARDTYPE=pc KEYTABLE=us **rd_LVM_LV=rootvg/lv_swap** rd_NO_DM rhgb quiet

In each example, the portion in bold shows that the GRUB has to detect two LVM devices with the names "root" and "swap" from the volume group "rootvg."

### Fix the problem

If the LVM device doesn't exist, either create it or remove the corresponding parameters from the GRUB configuration files. Then, try again to enable protection.

## A Site Recovery Mobility Service update finished with warnings (error code 151083)

The Site Recovery Mobility Service has many components, one of which is called the filter driver. The filter driver is loaded into system memory only during system restart. Whenever a Mobility Service update includes filter driver changes, the machine is updated but you still see a warning that some fixes require a restart. The warning appears because the filter driver fixes can take effect only when the new filter driver is loaded, which happens only during a restart.

> [!NOTE]
> This is only a warning. Existing replication continues to work even after the new agent update. You can choose to restart whenever you want the benefits of the new filter driver, but the old filter driver keeps working if you don't restart.
>
> Apart from the filter driver, the benefits of any other enhancements and fixes in the Mobility Service update take effect without requiring a restart.  

## Protection couldn't be enabled because the replica managed disk already exists, without expected tags, in the target resource group (error code 150161)

### Possible cause

This problem can occur if the virtual machine was previously protected, and when replication was disabled, the replica disk was not cleaned.

### Fix the problem

Delete the replica disk identified in the error message and retry the failed protection job.

## Next steps

[Replicate Azure virtual machines](site-recovery-replicate-azure-to-azure.md)
