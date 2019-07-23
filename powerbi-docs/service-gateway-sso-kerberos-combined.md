---
title: Prepare for single sign-on (SSO) - Kerberos
description: Configure your gateway with Kerberos to enable SSO from Power BI to on-premises data sources
author: mgblythe
ms.author: mblythe
manager: kfile
ms.reviewer: ''
ms.service: powerbi
ms.subservice: powerbi-gateways
ms.topic: conceptual
ms.date: 07/15/2019
LocalizationGroup: Gateways
---

# Prepare for Kerberos-based single sign-on (SSO) from Power BI service to on-premises data sources

Several items must be configured for Kerberos constrained delegation to work properly, including _Service Principal Names_ (SPN) and delegation settings on service accounts.

### Prerequisite 1: Install and configure the Microsoft on-premises data gateway

The on-premises data gateway supports an in-place upgrade, and _settings takeover_ of existing gateways.

### Prerequisite 2: Run the gateway Windows service as a domain account

In a standard installation, the gateway runs as a machine-local service account (specifically, _NT Service\PBIEgwService_), as shown in the following image:

![Screenshot of service account](media/service-gateway-sso-kerberos/service-account.png)

To enable Kerberos constrained delegation, the gateway must run as a domain account, unless your Azure Active Directory (Azure AD) instance is already synchronized with your local Active Directory instance (using Azure AD DirSync/Connect). To switch to a domain account, see [Change the gateway service account](/data-integration/gateway/service-gateway-service-account).

> [!NOTE]
> If Azure AD Connect is configured, and user accounts are synchronized, the gateway service doesn't need to perform local Azure AD lookups at runtime. You can use the local service SID (instead of requiring a domain account) for the gateway service. The Kerberos constrained delegation configuration steps outlined in this article are the same as that configuration. They are simply applied to the gateway's computer object in Azure AD, instead of the domain account.

### Prerequisite 3: Have domain admin rights to configure SPNs (SetSPN) and Kerberos constrained delegation settings

We don't recommended that a domain administrator temporarily or permanently allows rights to someone else to configure SPNs and Kerberos delegation without requiring domain admin rights. In the following section, we cover the recommended configuration steps in more detail.

## Configure Kerberos constrained delegation for the gateway and data source

As a domain administrator, configure an SPN for the gateway service domain account (if required), and configure delegation settings on the gateway service domain account.

### Configure an SPN for the gateway service account

First, determine whether an SPN was already created for the domain account used as the gateway service account:

1. As a domain administrator, launch **Active Directory Users and Computers**.

2. Right-click on the domain, select **Find**, and enter the account name of the gateway service account.

3. In the search result, right-click on the gateway service account and select **Properties**.

4. If the **Delegation** tab is visible on the **Properties** dialog, then an SPN was already created and you can jump ahead to the next subsection -- [configuring delegation settings](#configure-delegation-settings).

    If there is no **Delegation** tab on the **Properties** dialog box, you can manually create an SPN on that account. This adds the **Delegation** tab. Use the [setspn tool](https://technet.microsoft.com/library/cc731241.aspx) that comes with Windows (you need domain admin rights to create the SPN).

    For example, imagine the gateway service account is "PBIEgwTest\GatewaySvc", and the machine name with the gateway service running is called **Machine1**. To set the SPN for the gateway service account for that machine in this example, you would run the following command:

    ![Image of set SPN command](media/service-gateway-sso-kerberos/set-spn.png)

 With that step completed, we can move on to configuring delegation settings. Delegation settings can be configured for _either_ resource-based constrained Kerberos delegation or standard Kerberos constrained delegation. See the [constrained Kerberos delegation overview page](/windows-server/security/kerberos/kerberos-constrained-delegation-overview) for more information on the differences between the two approaches. Once you have decided which approach is most suitable, proceed to _either_ the [configure standard Kerberos constrained delegation](#FIX ME) _or_ [configure resource-based Kerberos constrained delegation](#FIX ME) subsection. Do not complete both subsections.

### Configure the gateway service account for standard Kerberos constrained delegation

> [!NOTE]
> Complete the steps in this section if you want to enable standard Kerberos constrained delegation. If you want to enable resource-based Kerberos constrained delegation complete the steps in the [Configure standard Kerberos constrained delegation](#FIX ME) subsection.

We will now set the delegation settings for the gateway service account. There are multiple tools you can use to perform these steps. Here, we'll use Active Directory Users and Computers, which is a Microsoft Management Console (MMC) snap-in to administer and publish information in the directory. It's available on domain controllers by default. You can also enable it through Windows Feature configuration on other machines.

We need to configure Kerberos constrained delegation with protocol transiting. With constrained delegation, you must be explicit about which services you want to delegate to. For example, only SQL Server or your SAP HANA server accepts delegation calls from the gateway service account. Note that if your gateway machine and your data source reside in different domains then you need to use [resource-based Kerberos constrained delegation](#FIXME).

This section assumes you have already configured SPNs for your underlying data sources (such as SQL Server, SAP HANA, Teradata, and Spark). To learn how to configure those data source server SPNs, refer to technical documentation for the respective database server. You can also see the heading *What SPN does your app require?* in the [My Kerberos Checklist](https://techcommunity.microsoft.com/t5/SQL-Server-Support/My-Kerberos-Checklist-8230/ba-p/316160) blog post.

In the following steps, we assume an on-premises environment with two machines: a gateway machine and a database server running SQL Server. For the sake of this example, we'll also assume the following settings and names:

* Gateway machine name: **PBIEgwTestGW**
* Gateway service account: **PBIEgwTest\GatewaySvc** (account display name: Gateway Connector)
* SQL Server data source machine name: **PBIEgwTestSQL**
* SQL Server data source service account: **PBIEgwTest\SQLService**

Here's how to configure the delegation settings:

1. With domain administrator rights, open **Active Directory Users and Computers**.

2. Right-click the gateway service account (**PBIEgwTest\GatewaySvc**), and select **Properties**.

3. Select the **Delegation** tab.

4. Select **Trust this computer for delegation to specified services only** > **Use any authentication protocol**.

5. Under **Services to which this account can present delegated credentials**, select **Add**.

6. In the new dialog box, select **Users or Computers**.

7. Enter the service account for the data source, for example, a SQL Server data source may have a service account like  **PBIEgwTest\SQLService**. Once the account has been added, select **OK**.

8. Select the SPN that you created for the database server. In our example, the SPN begins with **MSSQLSvc**. If you added both the FQDN and the NetBIOS SPN for your database service, select both. You might only see one.

9. Select **OK**. You should see the SPN in the list now.

    Optionally, you can select **Expanded** to show both the FQDN and NetBIOS SPN. The dialog box looks similar to the following if you selected **Expanded**. Select **OK**.

    ![Screenshot of Gateway Connector Properties dialog box](media/service-gateway-sso-kerberos/gateway-connector-properties.png)

### Configure the gateway service account for resource-based Kerberos constrained delegation

> [!NOTE]
> Complete the steps in this section if you want to enable resource-based Kerberos constrained delegation. If you want to enable standard Kerberos constrained delegation complete the steps in the [Configure standard Kerberos constrained delegation](#FIX ME) subsection.

In the following steps, we assume an on-premises environment with two machines in different domains: a gateway machine and a database server running SQL Server that has already been configured for Kerberos-based single sign-on (SSO). For the sake of this example, we also assume the following settings and names:

* Gateway machine name: **PBIEgwTestGW**
* Gateway service account: **PBIEgwTestFrontEnd\GatewaySvc** (account display name: Gateway Connector)
* SQL Server data source machine name: **PBIEgwTestSQL**
* SQL Server data source service account: **PBIEgwTestBackEnd\SQLService**

Given those example names and settings, use the following configuration steps:

1. Using **Active Directory Users and Computers**, which is a Microsoft Management Console (MMC) snap-in, on the domain controller for **PBIEgwTestFront-end** domain, make sure no delegation settings are applied for the gateway service account.

    ![Gateway connector properties](media/service-gateway-sso-kerberos-resource/gateway-connector-properties.png)

2. Using **Active Directory Users and Computers**, on the domain controller for **PBIEgwTestBack-end** domain, make sure no delegation settings are applied for the back-end service account. In addition, make sure that the "msDS-AllowedToActOnBehalfOfOtherIdentity" attribute for this account is also not set. You can find this attribute in the "Attribute Editor", as shown in the following image:

    ![SQL service properties](media/service-gateway-sso-kerberos-resource/sql-service-properties.png)

3. Create a group in **Active Directory Users and Computers**, on the domain controller for **PBIEgwTestBack-end** domain. Add the gateway service account to this group as shown in the following image. The image shows a new group called _ResourceDelGroup_ and the gateway service account **GatewaySvc** added to this group.

    ![Group properties](media/service-gateway-sso-kerberos-resource/group-properties.png)

4. Open a command prompt and run the following commands in the domain controller for **PBIEgwTestBack-end** domain to update the msDS-AllowedToActOnBehalfOfOtherIdentity attribute of the back-end service account:

    ```powershell
    $c = Get-ADGroup ResourceDelGroup
    Set-ADUser SQLService -PrincipalsAllowedToDelegateToAccount $c
    ```

5. You can verify that the update is reflected in the "Attribute Editor" tab in the properties for the back-end service account in **Active Directory Users and Computers.**

### Grant the gateway service account local policy rights on the gateway machine

On the machine running the gateway service (**PBIEgwTestGW** in our example), you must grant the gateway service account the local policy **Impersonate a client after authentication** and **Act as part of the operating system (SeTcbPrivilege)**. You can perform and verify this configuration with the Local Group Policy Editor (**gpedit**).

1. On the gateway machine, run: *gpedit.msc*.

2. Go to **Local Computer Policy** > **Computer Configuration** > **Windows Settings** > **Security Settings** > **Local Policies** > **User Rights Assignment**.

    ![Screenshot of Local Computer Policy folder structure](media/service-gateway-sso-kerberos/user-rights-assignment.png)

3. Under **User Rights Assignment**, from the list of policies, select **Impersonate a client after authentication**.

    ![Screenshot of Impersonate a client policy](media/service-gateway-sso-kerberos/impersonate-client.png)

    Right-click, and open **Properties**. Check the list of accounts. It must include the gateway service account (**PBIEgwTest\GatewaySvc**).

4. Under **User Rights Assignment**, from the list of policies, select **Act as part of the operating system (SeTcbPrivilege)**. Ensure that the gateway service account is included in the list of accounts as well.

5. Restart the **On-premises data gateway** service process.

If you're using SAP HANA, we recommend following these additional steps, which can yield a small performance improvement.

1. In the gateway installation directory, find and open this configuration file: *Microsoft.PowerBI.DataMovement.Pipeline.GatewayCore.dll.config*.

2. Find the *FullDomainResolutionEnabled* property, and change its value to *True*.

    ```xml
    <setting name=" FullDomainResolutionEnabled " serializeAs="String">
          <value>True</value>
    </setting>
    ```
## Complete datasource-specific configuration steps

SAP HANA and SAP BW have additional data-source specific configuration requirements and prerequisites that need to be met before you can establish an SSO connection through the gateway to these data sources. See [single sign-on (SSO) for SAP Hana - Kerberos](#FIXME) and 
[single sign-on (SSO) for SAP BW using CommonCryptoLib - Kerberos](#FIXME) for details. Note that it is also possible to [configure SAP BW for use with the gsskrb5/gx64krb5 SNC library](#FIXME), though library is not reccomended by Microsoft as it is no longer supported by SAP.
    
## Run a Power BI report

After completing all the configuration steps, you can use the **Manage Gateway** page in Power BI to configure the data source. Then, under **Advanced Settings**, enable SSO, and publish reports and datasets binding to that data source. 
 
![Screenshot of Advanced settings option](media/service-gateway-sso-kerberos/advanced-settings.png)

This configuration works in most cases. However, with Kerberos there can be different configurations depending on your environment. If the report still won't load, contact your domain administrator to investigate further.

   
