﻿<properties
 pageTitle="Submit jobs to an HPC Pack cluster in Azure | Microsoft Azure"
 description="Learn how to set up an on-premises computer to submit jobs to an HPC Pack cluster in Azure"
 services="virtual-machines"
 documentationCenter=""
 authors="dlepow"
 manager="timlt"
 editor=""
 tags="azure-resource-manager,azure-service-management,hpc-pack"/>
<tags
ms.service="virtual-machines"
 ms.devlang="na"
 ms.topic="article"
 ms.tgt_pltfrm="vm-multiple"
 ms.workload="big-compute"
 ms.date="09/28/2015"
 ms.author="danlep"/>

# Submit HPC jobs from an on-premises computer to an HPC Pack cluster in Azure

[AZURE.INCLUDE [learn-about-deployment-models](../../includes/learn-about-deployment-models-both-include.md)]

This article shows you how to configure an on-premises client computer running Windows to run HPC Pack job submission tools that communicate with an HPC Pack cluster in Azure over HTTPS. This provides a straightforward, flexible way for a variety of cluster users to submit jobs to a cloud-based HPC Pack cluster without needing to connect directly to the head node VM to
run job submission tools.

![Submit a job to a cluster in Azure][jobsubmit]

## Prerequisites

* **HPC Pack head node deployed in an Azure VM** - You can use
automated tools such as an [Azure quickstart template](https://azure.microsoft.com/documentation/templates/) or an [Azure PowerShell script](virtual-machines-hpcpack-cluster-powershell-script.md)
to deploy the head node and cluster, or you can deploy the cluster
manually in Azure as you would for an on-premises cluster. You will need the DNS
name of the head node and the credentials of a cluster administrator to
complete the steps in this article.

    If you deployed the head node manually, ensure that an HTTPS endpoint is configured in the VM. If it is not, set one up. See [How to Set Up Endpoints to a Virtual
Machine](virtual-machines-set-up-endpoints.md).

* **HPC Pack installation media** - The free installation package for the
latest version of HPC Pack (HPC Pack 2012 R2) is available from the
[Microsoft Download
Center](http://go.microsoft.com/fwlink/?LinkId=328024). Ensure that you
select the same version of HPC Pack that is installed on the head node
VM.

* **Client computer** - You'll need a Windows or Windows Server client computer that can run HPC Pack client utilities (see [system requirements](https://technet.microsoft.com/library/dn535781.aspx)). If you only want to use the HPC Pack web portal or REST API to submit jobs, you can use a client computer of your choice.


## Step 1: Install and configure the web components on the head node

To enable a REST interface to submit jobs to the cluster over HTTPS,
install and configure the HPC Pack web components on the HPC Pack head
node, if they are not already configured. You first install the web
components by running the HpcWebComponents.msi installation file. Then,
configure the components by running the HPC PowerShell script
**Set-HPCWebComponents.ps1**.

For detailed procedures, see [Install the Microsoft HPC Pack Web
Components](http://technet.microsoft.com/library/hh314627.aspx).

>[AZURE.TIP] If you use the [HPC Pack IaaS deployment script](virtual-machines-hpcpack-cluster-powershell-script.md) to create the cluster,
you can optionally install and configure the web web components as part of the deployment.

**To install the web components**

1. Connect to the head node VM by using the credentials of a cluster administrator.

2. From the HPC Pack Setup folder, run HpcWebComponents.msi on the head node.

3. Follow the steps in the wizard to install the web components

**To configure the web components**

1. On the head node, start HPC PowerShell as an administrator.

2. To change directory to the location of the configuration script, type the following command:

    ```
    cd $env:CCP_HOME\bin
    ```
3. To configure the REST interface and start the HPC Web Service, type the following command:

    ```
    .\Set-HPCWebComponents.ps1 –Service REST –enable
    ```

4. When prompted to select a certificate, choose the certificate that corresponds to the public DNS name of the head node (CN=&lt;*HeadNodeDnsName*&gt;.cloudapp.net).

    >[AZURE.NOTE] You need to select this certificate to submit jobs later to the head node from an on-premises computer. Don't select or configure a certificate that corresponds to the computer name of the head node in the Active Directory domain (for example, CN=*MyHPCHeadNode.HpcAzure.local*).

5. To configure the web portal for job submission, type the following command:

    ```
    .\Set-HPCWebComponents.ps1 –Service Portal -enable
    ```
6. After the script completes, stop and restart the HPC Job Scheduler Service by typing the following:

    ```
    net stop hpcscheduler
    net start hpcscheduler
    ```

## Step 2: Install the HPC Pack client utilities on an on-premises computer

If you haven't already done so, download a compatible version of the
HPC Pack setup files from the [Microsoft Download
Center](http://go.microsoft.com/fwlink/?LinkId=328024) to the client
computer. When you begin the installation, choose the setup option for the HPC Pack client utilities.

To use the HPC Pack client tools to submit jobs to the head node VM, you'll also need to export a certificate from the head node and install it on the client computer. You'll need the certificate to be in .CER format.

**To export the certificate from the head node**

1. On the head node, add the Certificates snap-in to a Microsoft Management Console for the Local Computer account. For steps to add the snap-in, see [Add the Certificates Snap-in to an MMC](https://technet.microsoft.com/library/cc754431.aspx).

2. In the console tree, expand **Certificates – Local Computer**, expand **Personal**, and then click **Certificates**.

3. Locate the certificate that you configured for the HPC Pack web components in [Step 1: Install and configure the web components on the head node](#step-1:-install-and-configure-the-web-components-on-the-head-node) (for example, named &lt;HeadNodeDnsName&gt;.cloudapp.net).

4. Right-click the certificate, click **All Tasks**, and then click **Export**.

5. In the Certificate Export Wizard, click **Next**, and ensure that **No, do not export the private key** is selected.

6. Follow the remaining steps of the wizard to export the certificate in DER encoded binary X.509 (.CER) format.


**To import the certificate on the client computer**


1. Copy the certificate that you exported from the head node to a folder on the client computer.

2. On the client computer, run certmgr.msc.

3. In Certificate Manager, expand **Certificates – Current user**, expand **Trusted Root Certification Authorities**, right-click **Certificates**, click **All Tasks**, and then click **Import**.

4. In the Certificate Import Wizard, click **Next** and follow the steps to import the certificate that your exported from the head node.



>[AZURE.SECURITY] You might see a security warning, because the certification authority on the head node will not be recognized by the client computer. For testing purposes you can ignore this warning and complete the certificate import.

## Step 3: Run test jobs on the cluster

To verify your configuration, try running jobs on the cluster in Azure
by using the on-premises computer that is running the HPC Pack client
utilities. For example, you can use HPC Pack GUI tools or command-line commands to submit jobs to the cluster. You can also use a web-based portal to submit jobs.


**To run job submission commands on the client computer**


1. On the client computer, start a Command prompt.

2. Type a sample command. For example, to list all jobs on the cluster, type the following

    ```
    job list /scheduler:https://<HeadNodeDnsName>.cloudapp.net /all
    ```
    
    >[AZURE.TIP] Use the full DNS name of the head node, not the IP address, in the scheduler URL. If you specify the IP address, you’ll see an error similar to "The server certificate needs to either have a valid chain of trust or to be placed in the trusted root store".

3. When prompted, type the user name (in the form &lt;DomainName&gt;\\&lt;UserName&gt;) and password of the HPC cluster administrator or another cluster user that you have configured. You can choose to store the credentials locally for more job operations.

    A list of jobs appears.


**To use HPC Job Manager on the client computer**

1. If you didn't previously store domain credentials for a cluster user on the client computer when you submitted the job, you can add the credentials in Credential Manager.

    a. In Control Panel on the client computer, start Credential Manager.

    b. Click **Windows Credentials**, and then click **Add a generic credential**.

    c. Specify the Internet address https://&lt;HeadNodeDnsName&gt;.cloudapp.net/HpcScheduler, and provide the user name (in the form &lt;DomainName&gt;\\&lt;UserName&gt;) and password of the HPC cluster administrator or another cluster user that you configured.

2. On the client computer, start HPC Job Manager.

3. In the **Select Head Node** dialog box, type the URL to the head node in Azure in the form  https://&lt;HeadNodeDnsName&gt;.cloudapp.net.

    HPC Job Manager opens and shows a list of jobs on the head node.

**To use the web portal running on the head node**

1. Start a web browser on the client computer, and type the following address:

    ```
    https://<HeadNodeDnsName>.cloudapp.net/HpcPortal
    ```
2. In the security dialog box that appears, type the domain credentials of the HPC cluster administrator. (You can also add other cluster users in different roles. For more information, see [Managing Cluster Users](https://technet.microsoft.com/library/ff919335.aspx).)

    The web portal opens to the job list view.

3. To submit a sample job that returns the string “Hello World” from the cluster, click **New job** in the left-hand navigation.

4. On the **New Job** page, under **From submission pages**, click **HelloWorld**. The job submission page appears.

5. Click **Submit**. If prompted, provide the domain credentials of the HPC cluster administrator. The job is submitted and the job ID appears on the **My Jobs** page.

6. To view the results of the job that you submitted, click the job ID, and then click **View Tasks** to view the command output (under **Output**).

## Next steps

* You can also submit jobs to the Azure cluster with the [HPC Pack REST API](http://social.technet.microsoft.com/wiki/contents/articles/7737.creating-and-submitting-jobs-by-using-the-rest-api-in-microsoft-hpc-pack-windows-hpc-server.aspx).

* If you want to submit cluster from a Linux client, see the Python sample in the [HPC Pack SDK](https://www.microsoft.com/download/details.aspx?id=47756).


<!--Image references-->
[jobsubmit]: ./media/virtual-machines-hpcpack-cluster-submit-jobs/jobsubmit.png
