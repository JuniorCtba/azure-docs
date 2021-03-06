---
title: Migration guide for Azure Stack Edge Pro FPGA to GPU physical device
description: This guide contains instructions to migrate workloads from an Azure Stack Edge Pro FPGA device to an Azure Stack Edge Pro GPU device.
services: databox
author: alkohli

ms.service: databox
ms.subservice: edge
ms.topic: tutorial
ms.date: 02/10/2021
ms.author: alkohli  
---
# Migrate workloads from an Azure Stack Edge Pro FPGA to an Azure Stack Edge Pro GPU

This article describes how to migrate workloads and data from an Azure Stack Edge Pro FPGA device to an Azure Stack Edge Pro GPU device. The migration procedure involves an overview of migration including a comparison between the two devices, migration considerations, detailed steps, and verification followed by cleanup.

<!--Azure Stack Edge Pro FPGA devices will reach end-of-life in February 2024. If you are considering new deployments, we recommend that you explore Azure Stack Edge Pro GPU devices for your workloads.-->

## About migration

Migration is the process of moving workloads and application data from one storage location to another. This entails making an exact copy of an organization’s current data from one storage device to another storage device - preferably without disrupting or disabling active applications - and then redirecting all input/output (I/O) activity to the new device. 

This migration guide provides a step-by-step walkthrough of the steps required to migrate data from an Azure Stack Edge Pro FPGA device to an Azure Stack Edge Pro GPU device. This document is intended for information technology (IT) professionals and knowledge workers who are responsible for operating, deploying, and managing Azure Stack Edge devices in the datacenter. 

In this article, the Azure Stack Edge Pro FPGA device is referred to as the *source* device and the Azure Stack Edge Pro GPU device is the *target* device. 

## Comparison summary

This section provides a comparative summary of capabilities between the Azure Stack Edge Pro GPU vs. the Azure Stack Edge Pro FPGA devices. The hardware in both the source and the target device is largely identical and differs only with respect to the hardware acceleration card and the storage capacity. 

|    Capability  | Azure Stack Edge Pro GPU (Target device)  | Azure Stack Edge Pro FPGA (Source device)|
|----------------|-----------------------|------------------------|
| Hardware       | Hardware acceleration: 1 or 2 Nvidia T4 GPUs <br> Compute, memory, network interface, power supply unit, power cord specifications are identical to the device with FPGA.  | Hardware acceleration: Intel Arria 10 FPGA <br> Compute, memory, network interface, power supply unit, power cord specifications are identical to the device with GPU.          |
| Usable storage | 4.19 TB <br> After reserving space for parity resiliency and internal use | 12.5 TB <br> After reserving space for internal use |
| Security       | Certificates |                                                     |
| Workloads      | IoT Edge workloads <br> VM workloads <br> Kubernetes workloads| IoT Edge workloads |
| Pricing        | [Pricing](https://azure.microsoft.com/pricing/details/azure-stack/edge/) | [Pricing](https://azure.microsoft.com/pricing/details/azure-stack/edge/)|

## Migration plan

To create your migration plan, consider the following information:

- Develop a schedule for migration. 
- When you migrate data, you may experience a downtime. We recommend that you schedule migration during a downtime maintenance window as the process is disruptive. You will set up and restore configurations in this downtime as described later in this document.
- Understand the total length of downtime and communicate it to all the stakeholders.
- Identify the local data that needs to be migrated from the source device. As a precaution, ensure that all the data on the existing storage has a recent backup. 


## Migration considerations 

Before you proceed with the migration, consider the following information: 

- An Azure Stack Edge Pro GPU device can't be activated against an Azure Stack Edge Pro FPGA resource. A new resource should be created for the Azure Stack Edge Pro GPU device as described in the [Create an Azure Stack Edge Pro GPU order](azure-stack-edge-gpu-deploy-prep.md#create-a-new-resource).
- The Machine Learning models deployed on the source device that used the FPGA will need to be changed for the target device with GPU. For help with the models, you can contact Microsoft Support. The custom models deployed on the source device that did not use the FPGA (used CPU only) should work as-is on the target device (using CPU).
- The IoT Edge modules deployed on the source device may require changes before these can be successfully deployed on the target device. 
- The source device supports NFS 3.0 and 4.1 protocols. The target device only supports NFS 3.0 protocol.
- The source device support SMB and NFS protocols. The target device supports storage via REST protocol using storage accounts in addition to SMB and NFS protocols for shares.
- The share access on the source device is via the IP address whereas the share access on the target device is via the device name.

## Migration steps at-a-glance

This table summarizes the overall flow for migration, describing the steps required for migration and the location where to take these steps.

| In this phase | Do this step| On this device |
|---------------|-------------|----------------|
| Prepare source device*       | 1. Record configuration data <br>2. Back up share data <br>3. Prepare IoT Edge workloads| Source device  |
| Prepare target device*       |1. Create a new order <br>2. Configure and activate| Target device  |
| Migrate data       | 1. Migrate data from shares <br>2. Redeploy IoT Edge workloads| Target device  |
| Verify data            |Verify migrated data |Target device  |
| Clean up, return              |Erase data and return| Source device  |

**The source and target devices can be prepared in parallel.*

## Prepare source device

The preparation includes that you identify the Edge cloud shares, Edge local shares, and the IoT Edge modules deployed on the device. 

### 1. Record configuration data

Do these steps on your source device via the local UI.

Record the configuration data on the *source* device. Use the [Deployment checklist](azure-stack-edge-gpu-deploy-checklist.md) to help you record the device configuration. During migration, you'll use this configuration information to configure the new target device. 

### 2. Back up share data

The device data can be of one of the following types:

- Data in Edge cloud shares
- Data in local shares

#### Data in Edge cloud shares

Edge cloud shares tier data from your device to Azure. Do these steps on your *source* device via the Azure portal. 

- Make a list of all the Edge cloud shares and users that you have on the source device.
- Make a list of all the bandwidth schedules that you have. You will recreate these bandwidth schedules on your target device.
- Depending on the network bandwidth available, configure bandwidth schedules on your device so as to maximize the data tiered to the cloud. This would minimize the local data on the device.
- Ensure that the shares are fully tiered to the cloud. This can be confirmed by checking the share status in the Azure portal.  

#### Data in Edge local shares

Data in Edge local shares stays on the device. Do these steps on your *source* device via the Azure portal. 

- Make a list of the Edge local shares that you have on the device.
- Given this is one-time migration of the data, create a copy of the Edge local share data to another on-premises server. You can use copy tools such as `robocopy` (SMB) or `rsync` (NFS) to copy the data. Optionally you may have already deployed a third-party data protection solution to back up the data in your local shares. The following third-party solutions are supported for use with Azure Stack Edge Pro FPGA devices:

    | Third-party software           | Reference to the solution                               |
    |--------------------------------|---------------------------------------------------------|
    | Cohesity                       | [https://www.cohesity.com/solution/cloud/azure/](https://www.cohesity.com/solution/cloud/azure/) <br> For details, contact Cohesity.          |
    | Commvault                      | [https://www.commvault.com/azure](https://www.commvault.com/azure) <br> For details, contact Commvault.          |
    | Veritas                        | [http://veritas.com/azure](http://veritas.com/azure) <br> For details, contact Veritas.   |
    | Veeam                          | [https://www.veeam.com/kb4041](https://www.veeam.com/kb4041) <br> For details, contact Veeam. |


### 3. Prepare IoT Edge workloads

- If you have deployed IoT Edge modules and are using FPGA acceleration, you may need to modify the modules before these will run on the GPU device. Follow the instructions in [Modify IoT Edge modules](azure-stack-edge-gpu-modify-fpga-modules-gpu.md). 

<!--- If you have deployed IoT Edge workloads, the configuration data is shared on a share on the device. Back up the data in these shares.-->


## Prepare target device

### 1. Create new order

You need to create a new order (and a new resource) for your *target* device. The target device must be activated against the GPU resource and not against the FPGA resource.

To place an order, [Create a new Azure Stack Edge resource](azure-stack-edge-gpu-deploy-prep.md#create-a-new-resource) in the Azure portal.


### 2. Set up, activate

You need to set up and activate the *target* device against the new resource you created earlier. 

Follow these steps to configure the *target* device via the Azure portal:

1. Gather the information required in the [Deployment checklist](azure-stack-edge-gpu-deploy-checklist.md). You can use the information that you saved from the source device configuration. 
1. [Unpack](azure-stack-edge-gpu-deploy-install.md#unpack-the-device), [rack mount](azure-stack-edge-gpu-deploy-install.md#rack-the-device) and [cable your device](azure-stack-edge-gpu-deploy-install.md#cable-the-device). 
1. [Connect to the local UI of the device](azure-stack-edge-gpu-deploy-connect.md).
1. Configure the network using a different set of IP addresses (if using static IPs) than the ones that you used for your old device. See how to [configure network settings](azure-stack-edge-gpu-deploy-configure-network-compute-web-proxy.md).
1. Assign the same device name as your old device and provide a DNS domain. See how to [configure device setting](azure-stack-edge-gpu-deploy-set-up-device-update-time.md).
1. Configure certificates on the new device. See how to [configure certificates](azure-stack-edge-gpu-deploy-configure-certificates.md).
1. Get the activation key from the Azure portal and activate the new device. See how to [activate the device](azure-stack-edge-gpu-deploy-activate.md).

You are now ready to restore the share data and deploy the workloads that you were running on the old device.

## Migrate data

You will now copy data from the source device to the Edge cloud shares and Edge local shares on your *target* device.

### 1. From Edge cloud shares

Follow these steps to sync the data on the Edge cloud shares on your target device:

1. [Add shares](azure-stack-edge-j-series-manage-shares.md#add-a-share) corresponding to the share names created on the source device. Make sure that while creating shares, **Select blob container** is set to **Use existing** option and then select the container that was used with the previous device.
1. [Add users](azure-stack-edge-j-series-manage-users.md#add-a-user) that had access to the previous device.
1. [Refresh the share](azure-stack-edge-j-series-manage-shares.md#refresh-shares) data from Azure. This pulls down all the cloud data from the existing container to the shares.
1. Recreate the bandwidth schedules to be associated with your shares. See [Add a bandwidth schedule](azure-stack-edge-j-series-manage-bandwidth-schedules.md#add-a-schedule) for detailed steps.


### 2. From Edge local shares

You may have deployed a third-party backup solution to protect the local shares data for your IoT workloads. You will now need to restore that data.

After the replacement device is fully configured, enable the device for local storage. 

Follow these steps to recover the data from local shares:

1. [Configure compute on the device](azure-stack-edge-gpu-deploy-configure-compute.md).
1. Add all the local shares on the target device. See the detailed steps in [Add a local share](azure-stack-edge-j-series-manage-shares.md#add-a-local-share).
1. Accessing the SMB shares on the source device will use the IP addresses whereas on the target device, you'll use device name. See [Connect to an SMB share on Azure Stack Edge Pro GPU](azure-stack-edge-j-series-deploy-add-shares.md#connect-to-an-smb-share). To connect to NFS shares on the target device, you'll need to use the new IP addresses associated with the device. See [Connect to an NFS share on Azure Stack Edge Pro GPU](azure-stack-edge-j-series-deploy-add-shares.md#connect-to-an-nfs-share). 

    If you copied over your share data to an intermediate server over SMB/NFS, you can copy this data over to shares on the target device. You can also copy the data over directly from the source device if both the source and the target device are *online*.

    If you had used a third-party software to back up the data in the local shares, you will need to run the recovery procedure provided by the data protection solution of choice. See references in the following table.

    | Third-party software           | Reference to the solution                               |
    |--------------------------------|---------------------------------------------------------|
    | Cohesity                       | [https://www.cohesity.com/solution/cloud/azure/](https://www.cohesity.com/solution/cloud/azure/) <br> For details, contact Cohesity.          |
    | Commvault                      | [https://www.commvault.com/azure](https://www.commvault.com/azure) <br> For details, contact Commvault. |
    | Veritas                        | [http://veritas.com/azure](http://veritas.com/azure) <br> For details, contact Veritas.   |
    | Veeam                          | [https://www.veeam.com/kb4041](https://www.veeam.com/kb4041) <br> For details, contact Veeam. |

### 3. Redeploy IoT Edge workloads

Once the IoT Edge modules are prepared, you will need to deploy IoT Edge workloads on your target device. If you face any errors in deploying IoT Edge modules, see:

- [Common issues and resolutions for Azure IoT Edge](../iot-edge/troubleshoot-common-errors.md). 
- [IoT Edge runtime errors](azure-stack-edge-gpu-troubleshoot.md#troubleshoot-iot-edge-errors).

## Verify data

After migration, verify that all the data has migrated and the workloads have been deployed on the target device.

## Erase data, return

After the data migration is complete, erase local data and return the source device. Follow the steps in [Return your Azure Stack Edge Pro device](azure-stack-edge-return-device.md).


## Next steps

[Learn how to deploy IoT Edge workloads on Azure Stack Edge Pro GPU device](azure-stack-edge-gpu-deploy-compute-module-simple.md)
