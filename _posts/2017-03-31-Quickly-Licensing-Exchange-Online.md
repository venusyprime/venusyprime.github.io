---
layout: post
title:  "Quickly Licensing Exchange Online Users using PowerShell"
date:   2017-03-31 12:00:00 +0000
categories: powershell exchange
---

So here's something that might be useful for other admins looking to move to Exchange Online - particularly if you're used to an on-premises environment, as I was.

To be fully set up for Exchange Online, a user needs a few things:

- They need a *-UsageLocation* set.
- They need a license assigning.
- If you're operating in a hybrid environment with Exchange 2010/2013, the mailbox needs migrating (this post will not cover that part). If you're not in a hybrid environment, assigning the license is enough to create the mailbox.
- For the user to be able to log in, they need their regional configuration (time zone) setting.
- In our case, I also needed to double check the domain used for the *UserPrincipalName* - a lot of our users were set up with a UPN matching our internal domain, when ideally they should login to Office 365 with the external domain that they're familiar with.

## Prerequisites

The most important thing here is the *Azure Active Directory cmdlets for PowerShell*, downloadable from [here](https://docs.microsoft.com/en-gb/powershell/azure/install-msonlinev1). We will be using the v1 cmdlets for this post, as I have not yet played around with the v2 ones.
(The V1 cmdlets are also referred to as the *Microsoft Online Services* cmdlets - hence the MSOL prefix they use).

This post assumes that you have user accounts synchronized from your local AD instance, because that's the scenario I'm familiar with. It also assumes your other domains have been set up properly in Office 365 and local AD, and that you have purchased the appropriate amount of licenses.

## The User Principal Name

This isn't mandatory, but does make it easier for users. The company I work for has several trading names, and an internal domain that does not match any of those trading names. Let's call the internal domain *internal.net*, and the domains we actually trade on as *companyA.com*, *companyB.com*.

>Side note: please don't visit any of these websites; they are being used as examples in place of real names and I cannot speak to the contents of them.

When users give out their email address, it ends in CompanyA.com or CompanyB.com. We set this up using an SMTP alias, tied to the Company attribute in AD. So using that same attribute, we can rewrite the user's UPN to match the domain they'd expect to see.

Fixing this only requires the regular Active Directory cmdlets.

``` powershell
$users = Get-ADUser -SearchBase "OU=Users,OU=Testing,DC=internal,DC=net" -Filter * -Properties Company

foreach ($user in $users) {
    Write-Verbose "Looking at $($user.UserPrincipalName)"
    $upnsplit = $user.UserPrincipalName.Split("@")
    switch ($user.Company) {
        "Company A" {$upnsuffix = "companya.com"}
        "Company B" {$upnsuffix = "companyb.com"}
        default {
            Write-Warning "$($user.UserPrincipalName) potentially has incorrect Company information."
            $upnsuffix = $upnsplit[1]
        }
        Write-Verbose "Setting $($user.UserPrincipalName) to $($upnsplit[0])@$upnsuffix"
        $user | Set-ADUser -UserPrincipalName "$($upnsplit[0])@$upnsuffix"
    }
}
```

Wait for a dirsync, and users will now have a login name that matches their expected domain. If the Company information does not match Company A or Company B (as was the case for a lot of service accounts we used, where they would only ever mail internally), it will set the UserPrincipalName to what it originally was.

## The Usage Location

The UsageLocation is how MS determines if a license can be applied to the account - as they cannot legally operate some Office 365 services in some territories.

If all of your users are in the same country, this is incredibly easy. If not already done, run `Connect-MSOLService` to connect PowerShell to your Office 365 instance. Then run:

``` powershell
Get-MSOLUser -All | Set-MSOLUser -UsageLocation GB
```

You're done. (See [ISO 3166-1 alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) for the full list of country codes and why you should use GB rather than UK).

You can pass normal AD users to the Set-MSOLUser command, which can be useful if most of your users are located in one country, but a group in their own OU is located in a different one.

## The License

After setting the user location, you can now assign a license. Run `Get-MSOLAccountSKU` to get the list of available licenses in your Office 365 tenant. Copy the relevant AccountSkuID and use it in the following commands. We'll use `contoso:EXCHANGESTANDARD` as an example.

If you only have one type of license to assign, and you want to assign it to all accounts that do not currently have a license, the easiest way is with:

``` powershell
Get-MsolUser -LicenseReconciliationNeededOnly | Set-MsolUserLicense -AddLicenses 'contoso:EXCHANGESTANDARD'
```

or

``` powershell
Get-MsolUser -UnlicensedUsersOnly | Set-MsolUserLicense -AddLicenses 'contoso:EXCHANGESTANDARD'
```

Use `-LicenseReconciliationNeededOnly` if you're migrating users - it'll show those that have been migrated without a license assigned. Use `-UnlicensedUsersOnly` otherwise - but be careful, especially in a hybrid environment, as this may show a lot of users that don't need licenses (users who have long since left the business; shared mailboxes that were set up as "User" mailboxes on-prem rather than "Shared" - if [converted to a shared mailbox](https://support.office.com/en-gb/article/Convert-a-user-mailbox-to-a-shared-mailbox-Admin-Help-2e122487-e1f5-4f26-ba41-5689249d93ba), they won't need a license if they use under 50GB of space).

To be more specific about license assignment, you can leverage your existing AD groups. For example:

``` powershell
Get-ADGroupMember "Branch Users" | Get-MSOLUser | Set-MsolUserLicense -AddLicenses 'contoso:EXCHANGESTANDARD'
```

You can also write a script to make sure they don't already have a license before assigning one, I might cover the one I wrote later.

## The Regional Configuration

If all of your users access their email via Outlook on their desktop, then you're pretty much done on the administration side after they've been migrated. However, if you have a large quantity of users reliant on Outlook Web Access, as we do, then not doing this can result in some unnecessary support calls.

When a user logs in for the first time to OWA for Office 365, they are prompted to set up their location and time zone settings. The location should be pre-filled in based on the UsageLocation above, but the time zone can be more confusing to users - we had a quantity that selected the first option on the list (Dateline Standard Time) rather than the GMT option, and then called to ask why the time was wrong on their emails.

To prevent this, we can use the `Set-MailboxRegionalConfiguration` command. This is part of the Exchange Online cmdlets rather than the MSOL ones, so we'll need to import the Exchange Online cmdlets first:

``` powershell
try {Get-Command Get-Mailbox -ErrorAction Stop | Out-Null}
catch {
    $Credential = Get-Credential -Message "Please enter your credentials for Exchange Online."
    $splat = @{
        ConfigurationName = "Microsoft.Exchange"
        ConnectionURI = "https://outlook.office365.com/powershell-liveid/"
        Credential = $Credential
        Authentication = "Basic"
        AllowRedirection = $true

    }
    $Session = New-PSSession @splat
    Import-PSSession $Session -DisableNameChecking
}
```

(if you're not familiar with the @splat syntax used, see [here](https://msdn.microsoft.com/en-us/powershell/reference/5.1/microsoft.powershell.core/about/about_splatting))

Now, we'll get the details for someone who did it properly:

``` powershell
Get-Mailbox GoodUser | Get-MailboxRegionalConfiguration

Identity             Language        DateFormat TimeFormat TimeZone
--------             --------        ---------- ---------- --------
GoodUser             en-GB           dd/MM/yyyy HH:mm      GMT Standard Time
```

And use that to fix everyone who didn't - or hasn't logged in yet.

``` powershell
$timezone = "GMT Standard Time"
Get-Mailbox | Get-MailboxRegionalConfiguration | where TimeZone -ne $timezone | Set-MailboxRegionalConfiguration -TimeZone $timezone
```

(It goes without saying, but be careful about international users; filter them out as close to the left side of the pipeline as is possible)