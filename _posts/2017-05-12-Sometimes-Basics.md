---
layout: post
title:  "Sometimes It's The Basics"
date:   2017-05-12 12:00:00 +0000
categories: powershell
---
I've been playing around with PowerShell DSC lately, improving on my home lab (although it's quite limited by the lack of compute resources/decent storage). Today, I wanted to get [credential encryption](https://msdn.microsoft.com/en-us/powershell/dsc/secureMOF) setup. So, I made a helper function to generate the certs, export files and write the certificate thumbprint to the console - of course, in production with an existing AD infrastructure, I'd use ADCS (and one of the purposes of this home lab is to learn how to set it up).

Here is the relevant helper function. **Do not copy and paste this yet**, for the reason I'm going to explain below:

```powershell
function GenerateDSCCertificate {
    [CmdletBinding()]
    param (
        [string]$PublicFilePath = "$psscriptroot\DSCCertficate.cer",
        [string]$PrivateFilePath = "$psscriptroot\DSCCertificate.pfx",
        [string]$SubjectName = "DSC Certificate for Home Lab",
        [Parameter(Mandatory=$true)]
        [SecureString]$Password
    )
    $WindowsMajorVersion = (Get-CimInstance Win32_OperatingSystem | Select-Object -ExpandProperty Version).Split(".")[0]
    #New-SelfSignedCertificate was upgraded on Windows 10.
    if ($WindowsMajorVersion -ge 10) {
        $cert = New-SelfSignedCertificate -Type DocumentEncryptionCertLegacyCsp -DnsName $SubjectName -HashAlgorithm SHA256
        $cert | Export-PfxCertificate -FilePath $PrivateFilePath -Password $Password
        $cert | Export-Certificate -FilePath $PublicFilePath
        $cert | Remove-Item -Force #remove the public cert from the local cert store
        Write-Output $cert.thumbprint
    }
    else {
        Write-Error "Cannot generate SelfSignedCertificate on this version of Windows. Please see this article: https://msdn.microsoft.com/en-us/powershell/dsc/secureMOF"
    }

}
```

Nice and simple. My authoring node is Windows 10, and my target nodes are Windows Server 2012 R2, so I made the decision to use Method 2 (again, it's a simple home lab), because that would be easier than having to load the New-SelfSignedCertificateEx script onto each node and copy the resulting public certificate back.

So, I then go to run a DSC configuration to generate a MOF. The actual configuration isn't too relevant, but it does include a component where it needs a domain credential to join the domain if not already joined. Here is the ConfigData block for that configuration.

``` powershell
$ConfigData = @{
    AllNodes = @(
        @{
            NodeName = "*"
            DomainName = "testing.internal.vei.space"
            DomainNetbiosName = "VEITest"
            CertificateFile = "$PSScriptRoot\DSCCertificate.cer"
            Thumbprint = "D5639ADD8A0F1765FB1103D1CDBF02139200443A"
            PSDSCAllowDomainUser = $true
        }
        @{
            NodeName = "Management"
        }
    )
}
```

Whenever I tried running this configuration, the same thing would happen:

``` powershell
ConvertTo-MOFInstance : System.ArgumentException error processing property 'Password' OF TYPE 'MSFT_Credential': Cannot 
load encryption certificate. The certificate setting 'C:\Users\Charles\OneDrive\Git\DSC_LabSetup\DSCCertificate.cer' 
does not represent a valid base-64 encoded certificate, nor does it represent a valid certificate by file, directory, 
thumbprint, or subject name.
At C:\WINDOWS\system32\WindowsPowerShell\v1.0\Modules\PSDesiredStateConfiguration\PSDesiredStateConfiguration.psm1:310 
char:13
+             ConvertTo-MOFInstance MSFT_Credential $newValue
+             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [Write-Error], InvalidOperationException
    + FullyQualifiedErrorId : FailToProcessProperty,ConvertTo-MOFInstance
```

So, what's the problem here? Some Googling and Binging gave nothing useful (if you're here because of a search engine, hello!). It took me some time to figure out the problem (during which I got it working by importing the public certificate into the authoring node cert store and passing the thumbprint instead), but let's take another look at those filepaths:

`[string]$PublicFilePath = "$psscriptroot\DSCCertficate.cer"`

`CertificateFile = "$PSScriptRoot\DSCCertificate.cer"`


I made a typo when setting up the helper function for the export! 🤦 I only noticed while I was tab-completing the certificate names to copy the PFX to the target node (thanks to the ability of Copy-Item to copy to a PSSession, I didn't even need to set up a SMB share with custom permissions due to the authoring node being a workgroup machine).

So yeah. Sometimes, it's the basics. Now I need to go set up a proper ADCS infrastructure and a pull server to make this sort of thing less hassle in the future.