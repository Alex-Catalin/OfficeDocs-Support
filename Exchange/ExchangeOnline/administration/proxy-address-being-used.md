---
title: Proxy address is already being used error message in Exchange Online
description: Describes an issue in which a mailbox isn't created for a user in Exchange Online.
author: simonxjx
audience: ITPro
ms.topic: troubleshooting
ms.custom: 
  - Exchange Online
  - CSSTroubleshoot
ms.author: v-six
manager: dcscontentpm
localization_priority: Normal
search.appverid: 
  - MET150
appliesto: 
  - Exchange Online
ms.date: 3/31/2022
---
# "Proxy address is already being used" or "Another object with the same value for property proxyAddresses already exists" error messages in Exchange Online

## Problem 1

When you try to create a new user in Exchange Online or when you try to assign an Exchange Online license to a Microsoft 365 user, you receive the following message:

> We are preparing the mailbox for this user.

However, the mailbox creation process doesn't complete successfully. If you try to access the properties of this user, you receive an error message that resembles the following:

> The proxy address "SMTP:\<user>@contoso.com" is already being used by "\<domain>.prod.outlook.com/Microsoft Exchange Hosted Organizations/contoso.onmicrosoft.com/\<forest>".
Choose another proxy address.

## Problem 2

You are trying to assign a new SMTP address to a user mailbox/shared mailbox/distribution group/M365 Group or you are trying to create a new user mailbox/shared mailbox/distribution group/M365 Group with that new address. At the moment when you attempt to save the changes, you receive an error message stating “An Azure Active Directory call was made to keep object in sync between Azure Active Directory and Exchange Online. However, it failed. Detailed error message: Another object with the same value for property proxyAddresses already exists. ConflictingObject: PublicFolder_<GUID value>”.

## Cause 1

An object that has the same proxy address already exists in Exchange Online.

## Cause 2  
  
For the error “An Azure Active Directory call was made to keep object in sync between Azure Active Directory and Exchange Online. However, it failed. Detailed error message: Another object with the same value for property proxyAddresses already exists. ConflictingObject: PublicFolder_<GUID value>”, the issue is caused by an orphan mail enable public folder object that exists only in Azure Active Directory and cannot be found in Exchange Online.

## Solution 1

Check for objects have the same proxy address, and then remove or change the proxy address of the object that's in conflict.
To determine which objects share the proxy address of a specified user, follow these steps:

1. Connect to Exchange Online by using a remote Windows PowerShell session. For more information about how to do this, see [Connect to Exchange Online using remote PowerShell](/powershell/exchange/connect-to-exchange-online-powershell).
2. Run the following command:

    ```powershell
    Get-EXORecipient -ResultSize unlimited | Where-Object {$_.EmailAddresses -match "user@contoso.onmicrosoft.com"} | Format-List Name, RecipientType, emailaddresses
    ```

    This command lists all mail recipients that have a type that matches the proxy address of a specified user. The duplicate proxy address may be associated with any of the following:
      - Site mailbox
      - Mail contact
      - Mail user
      - Mail-enabled distribution group
      - Group mailbox
      - Mail-enabled public folder

Only one proxy address at a time can be assigned to an object. After you determine which object is in conflict, you can remove or change the proxy address that's associated with that object.

To resolve this issue when creating shared mailboxes, see [Error when creating shared mailboxes](/microsoft-365/admin/email/resolve-issues-with-shared-mailboxes?view=o365-worldwide#error-when-creating-shared-mailboxes&preserve-view=true).

## Solution 2
  
1.	Remove the Conflicting SMTP Addresses from the Mail Enabled Public Folder (MEPF) in on-premises Exchange

2.	Wait for sync to run on the AAD Connect server

3.	Find the MEPF in AAD that is still using the SMTP address in question using below steps:

•	On the machine were AAD Connect server version 1.6 or greater is installed, open Windows PowerShell and run below cmdlets

Import-Module "C:\Program Files\Microsoft Azure Active Directory Connect\Tools\AdSyncTools.psm1" 

•	Enter O365 admin creds

$creds = Get-Credential

•	Set the ProxyAddress in conflict - Must include smtp: prefix but search is case-insensitive

$conflictingAddress = "smtp:user@contoso.com”
$mepfs = Get-ADSyncToolsAadObject -Credentials $creds -SyncObjectType   PublicFolder
$mepfs | Where-Object {$_.ProxyAddresses -icontains $conflictingAddress}

4.	Delete the MEPF by Removing the Synced MEPF by SourceAnchor

$conflictingObjSA= ‘SourceAnchor value in Base64 format'
 
Remove-ADSyncToolsAadObject -Credentials $creds -SyncObjectType PublicFolder -SourceAnchor $conflictingObjSA
 
5.	Immediately after deleting the MEPF, check to make sure it is gone by rerunning step 3. This time around, the output should be empty.

6.	Once confirmed the MEPF is gone from Azure Active Directory, immediately create the mailbox / assign the proxy address to a mailbox.  


## More information

For more information about primary addresses and proxy addresses, see [Add or remove email addresses for a mailbox](/exchange/recipients-in-exchange-online/manage-user-mailboxes/add-or-remove-email-addresses).

A reference to PowerShell AAD Connect cmdlets, can be found here https://docs.microsoft.com/en-us/azure/active-directory/hybrid/reference-connect-adsynctools.  
  
Still need help? Go to [Microsoft Community](https://answers.microsoft.com/).

