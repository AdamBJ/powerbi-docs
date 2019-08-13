---
title: Use Kerberos for single sign-on (SSO) to SAP BW using gx64krb5
description: Configure your SAP BW server to enable SSO from Power BI service using gx64krb5
author: mgblythe
ms.author: mblythe
manager: kfile
ms.reviewer: ''
ms.service: powerbi
ms.subservice: powerbi-gateways
ms.topic: conceptual
ms.date: 08/01/2019
LocalizationGroup: Gateways
---
> [!NOTE]
> Complete the steps on this page in addition to the steps in [Prepare for single sign-on (SSO) - Kerberos](TODO link) before attempting to refresh a SAP BW-based report that uses Kerberos SSO.

## Configure SAP BW for SSO using gx64krb5

Now that you have completed the configuration steps in [prepare for Kerberos constrained delegation](#prepare-for-kerberos-constrained-delegation) up to "Complete datasource-specific configuration", you can configure your SAP Business Warehouse (SAP BW) server for single sign-on (SSO) using the gateway.

> [!NOTE]
> Microsoft reccomends the use of CommonCryptoLib as your SNC library. SAP no longer offers support for gx64krb5/gsskrb5 and the steps required to configure it for use with the gateway are significantly more complex compared to CommonCryptoLib.

This guide attempts to be as comprehensive as possible. If you've already completed some of these steps, you can skip them. For example, you might have already created a service user for your SAP BW server and mapped an SPN to it, or you might have already installed the `gsskrb5` or `gx64krb5` library on the SAP BW server.

### Set up gx64krb5/gsskrb5 on gateway machine and the SAP BW server

> [!NOTE]
> `gx64krb5` and `gsskrb5` are no longer actively supported by SAP. For more information, see [SAP Note 352295](https://launchpad.support.sap.com/#/notes/352295). Also note that `gx64krb5` doesn't allow for SSO connections from the data gateway to SAP BW Message Servers. Only connections to SAP BW Application Servers are possible.

`gx64krb5` \ `gsskrb5` must be in use by both the client and server to complete an SSO connection through the gateway, i.e. both the client and server must be using the same SNC library.

1. Download `gx64krb5` from [SAP Note 2115486](https://launchpad.support.sap.com/) (SAP s-user required). Ensure you have at least version 1.0.11.x. Only download gsskrb5 (the 32-bit version of the library) if your SAP BW server requires a 32-bit .dll; gateway requires a 64-bit .dll.

1. Put `gx64krb5` in a location on your gateway machine that is accessible by your gateway instance (and also by the SAP GUI if you want to test the SSO connection by using SAP Logon). Both the gateway Service User and the Active Directory (AD) users that the Service User will impersonate need read and execute permissions for the .dll. We recommend granting permissions on the .dll to the Authenticated Users group. For testing purposes, you can also explicitly grant these permissions to both the gateway Service User and the Active Directory user you'll use to test.

1. If your BW server has not already been configured for SSO using gx64krb5/gsskrb5, put another copy on your SAP BW server machine in a location accessible by the SAP BW server. 

1. On the client and server machines, set the `SNC_LIB` or `SNC_LIB_64` environment variables. If you use gsskrb5, set the `SNC_LIB` variable to the absolute path of gsskrb5.dll. If you use gx64krb5, set the `SNC_LIB_64` variable to the absolute path of gx64krb5.dll.

### Create a SAP BW service user and enable SNC communication

Complete this section if you haven't already configured a Service User for your SAP BW server. Note that the rest of this guide assumes that the SNC name of the SAP BW server starts with "p:", e.g. p:\<the SAP BW service user you've created\>.

1. On an Active Directory Domain Controller server, create a service user (initially just a plain Active Directory user) for your SAP BW Application Server in your Active Directory environment. Then assign an SPN to it.

    SAP recommends starting the SPN with `SAP/`, but it should also be possible to use other prefixes, such as `HTTP/`. What comes after the `SAP/` is up to you; one option is to use the SAP BW server's service user's username. For example, if you create `BWServiceUser@\<DOMAIN\>` as your service user, you can use the SPN `SAP/BWServiceUser`. One way to set the SPN mapping is the setspn command. For example, to set the SPN on the service user we just created, you would run the following command from a cmd window on a Domain Controller machine: `setspn -s SAP/ BWServiceUser DOMAIN\ BWServiceUser`. For more information, see the SAP BW documentation.

1. Give the service user access to your SAP BW Application Server:

    1. On the SAP BW server machine, add the Service User to the Local Admin group. Open the Computer Management program and double-click the Local Admin group for your server.

        ![Screenshot of Computer Management program](media/service-gateway-sso-kerberos/computer-management.png)

    1. Double-click the Local Admin group, and select **Add** to add your service user to the group. Select **Check Names** to ensure you've entered the name correctly. Select **OK**.

1. Set the SAP BW server's Service User as the user that starts the SAP BW server service on the SAP BW server machine.

    1. Open **Run**, and enter "Services.msc". Look for the service corresponding to your SAP BW Application Server instance. Right-click it, and select **Properties**.

        ![Screenshot of Services, with Properties highlighted](media/service-gateway-sso-kerberos/server-properties.png)

    1. Switch to the **Log on** tab, and change the user to your SAP BW service user. Enter the user's password, and select **OK**.

1. Sign in to your server in SAP Logon and set the following profile parameters by using the RZ10 transaction:

    1. Set the snc/identity/as profile parameter to p:\<the SAP BW service user you've created\>, such as p:BWServiceUser@MYDOMAIN.COM. Note the p: that precedes the service user's UPN. It's not p:CN= like when Common Crypto Lib is used as the SNC library.

    1. Set the snc/gssapi\_lib profile parameter to \<path to gsskrb5.dll/gx64krb5.dll on the BW server machine (the library you'll use depends on OS bitness)\>. Remember to put the library in a location the SAP BW Application Server can access.

    1. Also set the following additional profile parameters, changing the values as required to fit your needs. Note that the last five options enable clients to connect to the SAP BW server by using SAP Logon without having SNC configured.

        | **Setting** | **Value** |
        | --- | --- |
        | snc/data\_protection/max | 3 |
        | snc/data\_protection/min | 1 |
        | snc/data\_protection/use | 9 |
        | snc/accept\_insecure\_cpic | 1 |
        | snc/accept\_insecure\_gui | 1 |
        | snc/accept\_insecure\_r3int\_rfc | 1 |
        | snc/accept\_insecure\_rfc | 1 |
        | snc/permit\_insecure\_start | 1 |

    1. Set the property snc/enable to 1.

1. After setting these profile parameters, open the SAP Management Console on the server machine, and restart the SAP BW instance. If the server won't start, confirm that you've set the profile parameters correctly. For more on profile parameter settings, see the [SAP documentation](https://help.sap.com/saphelp_nw70ehp1/helpdata/en/e6/56f466e99a11d1a5b00000e835363f/frameset.htm). You can also consult the troubleshooting information later in this section if you encounter problems.

### Map a SAP BW user to an Active Directory user

If you haven't already, map an Active Directory user to an SAP BW Application Server user, and test the SSO connection in SAP Logon.

1. Sign in to your SAP BW server by using SAP Logon. Run transaction SU01.

1. For **User**, enter the SAP BW user you want to enable SSO connections for (in the previous screenshot, we're setting permissions for BIUSER). Select the **Edit** icon (the image of a pen) near the top left of the SAP Logon window.

    ![Screenshot of SAP BW User maintenance screen](media/service-gateway-sso-kerberos/user-maintenance.png)

1. Select the **SNC** tab. In the SNC name input box, enter p:\<your Active Directory user\>@\<your domain\>. Note the mandatory p: that must precede the Active Directory user's UPN. The Active Directory user you specify should belong to the person or organization for whom you want to enable SSO access to the SAP BW Application Server. For example, if you want to enable SSO access for the user testuser\@TESTDOMAIN.COM, enter p:testuser@TESTDOMAIN.COM.

    ![Screenshot of SAP BW Maintain users screen](media/service-gateway-sso-kerberos/maintain-users.png)

1. Select the **Save** icon (the image of a floppy disk) near the top left of the screen.

### Test sign-in by using SSO

Verify that you can sign in to the server using SAP Logon through SSO as the Active Directory user for whom you've just enabled SSO access.

1. As the Active Directory user you just enabled SSO access for, sign in to a machine on which SAP Logon is installed. Launch SAP Logon, and create a new connection.

1. In the **Create New System Entry** screen, select **User Specified System** > **Next**.

    ![Screenshot of Create New System Entry screen](media/service-gateway-sso-kerberos/new-system-entry.png)

1. Fill in the appropriate details on the next screen, including the application server, instance number, and system ID. Then select **Finish**.

1. Right-click the new connection and select **Properties**. Select the **Network** tab. In the **SNC Name** text box, enter p:\<the SAP BW service user's UPN\>, such as p:BWServiceUser@MYDOMAIN.COM. Then select **OK**.

    ![Screenshot of System Entry Properties screen](media/service-gateway-sso-kerberos/system-entry-properties.png)

1. Double-click the connection you just created to attempt an SSO connection to your SAP BW server. If this connection succeeds, proceed to the next step. Otherwise, review the earlier steps in this document to make sure they've been completed correctly, or review the troubleshooting section below. Note that if you can't connect to the SAP BW server via SSO in this context you won't be able to connect to the SAP BW server using SSO in the gateway context.

### Troubleshoot gx64krb5/gsskrb5 configuration

If you encounter any problems, follow these steps to troubleshoot the gx64krb5/gsskrb5 installation and SSO connections from the SAP Logon.

* Viewing the server logs (…work\dev\_w0 on the server machine) can be helpful in troubleshooting any errors you encounter in completing the gx64krb5/gsskrb5 setup steps. This is particularly true if the SAP BW server won't start after the profile parameters have been changed.

* If you're unable to start the SAP BW service due to a logon failure, you might have provided the wrong password when setting the SAP BW "start-as" user. Verify the password by logging in to a machine in your Active Directory environment as the SAP BW service user.

* If you get errors about SQL credentials preventing the server from starting, verify that you've granted the service user access to the SAP BW database.

* You might get the following message: "(GSS-API) specified target is unknown or unreachable." This usually means you have the wrong SNC name specified. Make sure to use "p:" only, not "p:CN=" or anything else in the client application, other than the service user's UPN.

* You might get the following message: "(GSS-API) An invalid name was supplied." Make sure "p:" is in the value of the server's SNC identity profile parameter.

* You might get the following message: "(SNC error) the specified module could not be found." This is usually caused by putting the `gsskrb5.dll/gx64krb5.dll` somewhere that requires elevated privileges (administrator rights) to access.

### Add registry entries to the gateway machine

Add required registry entries to the registry of the machine that the gateway is installed on. Here are the commands to run:

1. REG ADD HKLM\SOFTWARE\Wow6432Node\SAP\gsskrb5 /v ForceIniCredOK /t REG\_DWORD /d 1 /f

1. REG ADD HKLM\SOFTWARE\SAP\gsskrb5 /v ForceIniCredOK /t REG\_DWORD /d 1 /f

### Add a new SAP BW Application Server data source to the Power BI service

Add the SAP BW data source to your gateway by following the instructions earlier in this article on [running a report](#run-a-power-bi-report).

1. In the data source configuration window, enter the Application Server's **Hostname**, **System Number**, and **client ID**, as you would to sign in to your SAP BW server from Power BI Desktop.

1. In the **SNC Partner Name** field, enter p: \<the SPN you mapped to your SAP BW service user\>. For example, if the SPN is SAP/BWServiceUser@MYDOMAIN.COM, you should enter p:SAP/BWServiceUser@MYDOMAIN.COM in the **SNC Partner Name** field.

1. For the SNC Library, select **SNC_LIB** or **SNC_LIB_64**. Make sure that SNC_LIB_64 on the gateway machine points to gx64krb5.dll. Alternatively, you can select the "Custom" option and provide the absolute path the gx64krb5.dll (on the gateway machine).

1. If you have selected **Windows** for **Authentication Method**, the **Username** and **Password** should be the username and password of an Active Directory user with permission to sign in to the SAP BW server with SSO. In other words, these should belong to an Active Directory user who has been mapped to a SAP BW user through the SU01 transaction. If you have selected **Basic**, the **Username** and **Password** should be set to a SAP BW user's name and password, respectively. These credentials are only used if the **Use SSO via Kerberos for DirectQuery queries** box is not checked.

1. Select the **Use SSO via Kerberos for DirectQuery queries** box, and select **Apply**. If the test connection is not successful, verify that the previous setup and configuration steps were completed correctly.

    The gateway always uses the typed-in credentials to establish a test connection to the server, and to do scheduled refreshes of import-based reports. The gateway only attempts to establish an SSO connection if the **Use SSO via Kerberos for DirectQuery queries** is selected, and the user is accessing a direct query-based report or dataset.


### Troubleshoot gateway connectivity issues

1. Check the gateway logs. Open the gateway configuration application, and select **Diagnostics** > **Export logs**. The most recent errors are at the bottom of any log files you examine.

    ![Screenshot of On-premises data gateway application, with Diagnostics highlighted](media/service-gateway-sso-kerberos/gateway-diagnostics.png)

1. Turn on SAP BW tracing, and review the generated log files. There are several different types of SAP BW tracing available. Consult the SAP documentation for more information.

## Errors from an insufficient Kerberos configuration

If the underlying database server and gateway are not configured properly for Kerberos constrained delegation, you might receive the following error message about failing to load data:

![Screenshot of error message](media/service-gateway-sso-kerberos/load-data-error.png)

The technical details associated with the error message (DM_GWPipeline_Gateway_ServerUnreachable) might look like the following:

![Screenshot of error message technical details](media/service-gateway-sso-kerberos/server-unreachable.png)

The result is that the gateway can't impersonate the originating user properly, and the database connection attempt failed.

## Next steps

For more information about the **on-premises data gateway** and **DirectQuery**, check out the following resources:

* [What is an on-premises data gateway?](/data-integration/gateway/service-gateway-getting-started)
* [DirectQuery in Power BI](desktop-directquery-about.md)
* [Data sources supported by DirectQuery](desktop-directquery-data-sources.md)
* [DirectQuery and SAP BW](desktop-directquery-sap-bw.md)
* [DirectQuery and SAP HANA](desktop-directquery-sap-hana.md)
