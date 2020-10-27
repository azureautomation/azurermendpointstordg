AzureRM-EndpointsToRDG
======================

            
**Create **an RDG file for all your ARM based VMs.** *** *

Creates an RDG file for Remote Desktop Connection Manager continaing all your Azure ARM Endpoints. Script can use internal IP addresses or External IP Addresses if VM contains External IP.


Script cycles through all subscriptions associated with your logged on account and creates an RDG file for all VMs on all subscriptions that you can connect to
Uses Public IP Address if it exists. Uses Private IP Address in there is no Public IP
if Hosts are behind a Load Balancer on a custom port, RDG file will have to be manually edited where port is not 3389


Multiple Nics not supported.
RDG file will be output to the location specified in the $wd variable (Default: Desktop)



Execute from Powershell or ISE. RDG file is created on your desktop by default** *** *

**


PowerShell
Edit|Remove
powershell
$wd='$env:USERPROFILE\Desktop'
SL $wd
clear-host

#Login to Azure
if (-not ($loggedgin))
    {
    Write-host 'Logging in to Azure' -fo Yellow
    # *** Enter ID and password below for automatic logon to Azure ***
    $userID =''
    $password ='' 
    if ($userID -eq '')
        {
            $loggedgin= Login-AzureRmAccount
        }
        Else
        {
            $pass=$password | ConvertTo-SecureString '' -AsPlainText -Force
            $Cred = New-Object System.Management.Automation.PSCredential($userID, $pass)
            $loggedgin= Login-AzureRmAccount -Credential $Cred
        }

    }


$useExternalhosts = Read-Host 'Use External IP Addresses if present?(Y/N)?' 
Switch ($useExternalhosts)
    {
        'y' {$useExternalhosts =$true}
        'n' {$useExternalhosts =$false}
        default {$useExternalhosts =$true}
    }
Write-host 'External Hosts Selected: $useExternalhosts'

$fileName=(Get-AzureRmTenant).Domain+'.rdg'
$path=(get-location).Path
$fullPath='$path\$fileName'

function Write-File
    {
        Param([string]$FileString)
        Add-Content $fileName -Value $FileString
    }


Write-host 'Creating RDG File: $fileName' -ForegroundColor Yellow 
$header=@'
<?xml version='1.0' encoding='utf-8'?>
<RDCMan programVersion='2.7' schemaVersion='3'>
  <file>
    <credentialsProfiles />
    <properties>
      <expanded>True</expanded>
      <name>Azure-VMs</name>
    </properties>
    <connectionSettings inherit='None'>
      <connectToConsole>True</connectToConsole>
      <workingDir>c:\windows\system32</workingDir>
      <port>3389</port>
      <loadBalanceInfo />
    </connectionSettings>
    <localResources inherit='None'>
      <audioRedirection>Client</audioRedirection>
      <audioRedirectionQuality>Dynamic</audioRedirectionQuality>
      <audioCaptureRedirection>DoNotRecord</audioCaptureRedirection>
      <keyboardHook>FullScreenClient</keyboardHook>
      <redirectClipboard>True</redirectClipboard>
      <redirectDrives>False</redirectDrives>
      <redirectDrivesList>
        <item>D:\</item>
      </redirectDrivesList>
      <redirectPrinters>False</redirectPrinters>
      <redirectPorts>False</redirectPorts>
      <redirectSmartCards>True</redirectSmartCards>
      <redirectPnpDevices>False</redirectPnpDevices>
    </localResources>
    <displaySettings inherit='None'>
      <liveThumbnailUpdates>True</liveThumbnailUpdates>
      <allowThumbnailSessionInteraction>False</allowThumbnailSessionInteraction>
      <showDisconnectedThumbnails>True</showDisconnectedThumbnails>
      <thumbnailScale>1</thumbnailScale>
      <smartSizeDockedWindows>True</smartSizeDockedWindows>
      <smartSizeUndockedWindows>True</smartSizeUndockedWindows>
    </displaySettings>
'@
set-content $fullpath $header

$subs=Get-AzureRmSubscription
foreach ($sub in $Subs)
{
Select-AzureRmSubscription -SubscriptionName $sub.SubscriptionName | Out-Null
$nodename =$sub.SubscriptionName
Write-host 'Processing Subscription: $nodename' -ForegroundColor Cyan
#Clear-host
Write-host 'Enumerating VMs. Please wait... ' -ForegroundColor Yellow
$vms=Get-AzureRmVM
Write-host 'Enumerating Network Interfaces. Please wait... ' -ForegroundColor Yellow
$nics=Get-AzureRmNetworkInterface
Write-host 'Enumerating Public IP Addresses. Please wait... ' -ForegroundColor Yellow
$PIPs = Get-AzureRmPublicIpAddress | ? {$_.IpConfiguration.id -match 'networkInterfaces'} | Select IpConfiguration, IpAddress


$AzureList=@()
$vms | %{
    $vm=$_.Name
    $nic = $nics | ?{$_.VirtualMachine.id -match $vm} 

    #Check if it has a public IP Address
    $Pip=$null
    $Pip=$pips |? {$_.IpConfiguration.id -match $nic.id}
    $PrivateIpAddress=$nic.IpConfigurations.PrivateIpAddress
    Write-host '$vm`t $PrivateIpAddress' -NoNewline -fo Yellow

    $azureObject = New-Object PSObject
    $azureObject | Add-Member -MemberType NoteProperty -Name 'Name' -Value $vm
    $azureObject | Add-Member -MemberType NoteProperty -Name 'IPAddress' -Value $PrivateIpAddress
    $azureObject | Add-Member -MemberType NoteProperty -Name 'PublicIPAddress' -Value $Pip.IpAddress
    Write-host 
    $AzureList+=$azureObject
    }

$content=@'
    <group>
      <properties>
        <expanded>False</expanded>
        <name>$nodename</name>
      </properties>
'@

add-content $fullpath $content

foreach ($server in $azureList){
    $displayName =$server.Name
        if ($server.PublicIPAddress -and $useExternalhosts -eq $true)
            {
            $IP=$server.PublicIPAddress
            }
        Else{
            $IP=$server.IPAddress
            }

    $port='3389'
    Write-host 'Adding Server: $displayName`t$IP'

$serverData=@'
        <server>
        <properties>
            <displayName>$displayName</displayName>
            <name>$IP</name>
        </properties>
        <connectionSettings inherit='None'>
            <connectToConsole>True</connectToConsole>
            <startProgram />
            <port>3389</port>
            <loadBalanceInfo />
        </connectionSettings>
        </server>
'@
    Add-content $fullpath $serverData
}

Add-content $fullpath @'
   </group>
'@
}
Add-content $fullpath @'
  </file>
  <connected />
  <favorites />
  <recentlyUsed />
</RDCMan>
'@
Write-host 'Process Complete. RDG file has been output to $fullPath' -ForegroundColor Green
start $fullPath


$wd='$env:USERPROFILE\Desktop' 
SL $wd 
clear-host 
 
#Login to Azure 
if (-not ($loggedgin)) 
    { 
    Write-host 'Logging in to Azure' -fo Yellow 
    # *** Enter ID and password below for automatic logon to Azure *** 
    $userID ='' 
    $password =''  
    if ($userID -eq '') 
        { 
            $loggedgin= Login-AzureRmAccount 
        } 
        Else 
        { 
            $pass=$password | ConvertTo-SecureString '' -AsPlainText -Force 
            $Cred = New-Object System.Management.Automation.PSCredential($userID, $pass) 
            $loggedgin= Login-AzureRmAccount -Credential $Cred 
        } 
 
    } 
 
 
$useExternalhosts = Read-Host 'Use External IP Addresses if present?(Y/N)?'  
Switch ($useExternalhosts) 
    { 
        'y' {$useExternalhosts =$true} 
        'n' {$useExternalhosts =$false} 
        default {$useExternalhosts =$true} 
    } 
Write-host 'External Hosts Selected: $useExternalhosts' 
 
$fileName=(Get-AzureRmTenant).Domain+'.rdg' 
$path=(get-location).Path 
$fullPath='$path\$fileName' 
 
function Write-File 
    { 
        Param([string]$FileString) 
        Add-Content $fileName -Value $FileString 
    } 
 
 
Write-host 'Creating RDG File: $fileName' -ForegroundColor Yellow  
$header=@' 
<?xml version='1.0' encoding='utf-8'?> 
<RDCMan programVersion='2.7' schemaVersion='3'> 
  <file> 
    <credentialsProfiles /> 
    <properties> 
      <expanded>True</expanded> 
      <name>Azure-VMs</name> 
    </properties> 
    <connectionSettings inherit='None'> 
      <connectToConsole>True</connectToConsole> 
      <workingDir>c:\windows\system32</workingDir> 
      <port>3389</port> 
      <loadBalanceInfo /> 
    </connectionSettings> 
    <localResources inherit='None'> 
      <audioRedirection>Client</audioRedirection> 
      <audioRedirectionQuality>Dynamic</audioRedirectionQuality> 
      <audioCaptureRedirection>DoNotRecord</audioCaptureRedirection> 
      <keyboardHook>FullScreenClient</keyboardHook> 
      <redirectClipboard>True</redirectClipboard> 
      <redirectDrives>False</redirectDrives> 
      <redirectDrivesList> 
        <item>D:\</item> 
      </redirectDrivesList> 
      <redirectPrinters>False</redirectPrinters> 
      <redirectPorts>False</redirectPorts> 
      <redirectSmartCards>True</redirectSmartCards> 
      <redirectPnpDevices>False</redirectPnpDevices> 
    </localResources> 
    <displaySettings inherit='None'> 
      <liveThumbnailUpdates>True</liveThumbnailUpdates> 
      <allowThumbnailSessionInteraction>False</allowThumbnailSessionInteraction> 
      <showDisconnectedThumbnails>True</showDisconnectedThumbnails> 
      <thumbnailScale>1</thumbnailScale> 
      <smartSizeDockedWindows>True</smartSizeDockedWindows> 
      <smartSizeUndockedWindows>True</smartSizeUndockedWindows> 
    </displaySettings> 
'@ 
set-content $fullpath $header 
 
$subs=Get-AzureRmSubscription 
foreach ($sub in $Subs) 
{ 
Select-AzureRmSubscription -SubscriptionName $sub.SubscriptionName | Out-Null 
$nodename =$sub.SubscriptionName 
Write-host 'Processing Subscription: $nodename' -ForegroundColor Cyan 
#Clear-host 
Write-host 'Enumerating VMs. Please wait... ' -ForegroundColor Yellow 
$vms=Get-AzureRmVM 
Write-host 'Enumerating Network Interfaces. Please wait... ' -ForegroundColor Yellow 
$nics=Get-AzureRmNetworkInterface 
Write-host 'Enumerating Public IP Addresses. Please wait... ' -ForegroundColor Yellow 
$PIPs = Get-AzureRmPublicIpAddress | ? {$_.IpConfiguration.id -match 'networkInterfaces'} | Select IpConfiguration, IpAddress 
 
 
$AzureList=@() 
$vms | %{ 
    $vm=$_.Name 
    $nic = $nics | ?{$_.VirtualMachine.id -match $vm}  
 
    #Check if it has a public IP Address 
    $Pip=$null 
    $Pip=$pips |? {$_.IpConfiguration.id -match $nic.id} 
    $PrivateIpAddress=$nic.IpConfigurations.PrivateIpAddress 
    Write-host '$vm`t $PrivateIpAddress' -NoNewline -fo Yellow 
 
    $azureObject = New-Object PSObject 
    $azureObject | Add-Member -MemberType NoteProperty -Name 'Name' -Value $vm 
    $azureObject | Add-Member -MemberType NoteProperty -Name 'IPAddress' -Value $PrivateIpAddress 
    $azureObject | Add-Member -MemberType NoteProperty -Name 'PublicIPAddress' -Value $Pip.IpAddress 
    Write-host  
    $AzureList+=$azureObject 
    } 
 
$content=@' 
    <group> 
      <properties> 
        <expanded>False</expanded> 
        <name>$nodename</name> 
      </properties> 
'@ 
 
add-content $fullpath $content 
 
foreach ($server in $azureList){ 
    $displayName =$server.Name 
        if ($server.PublicIPAddress -and $useExternalhosts -eq $true) 
            { 
            $IP=$server.PublicIPAddress 
            } 
        Else{ 
            $IP=$server.IPAddress 
            } 
 
    $port='3389' 
    Write-host 'Adding Server: $displayName`t$IP' 
 
$serverData=@' 
        <server> 
        <properties> 
            <displayName>$displayName</displayName> 
            <name>$IP</name> 
        </properties> 
        <connectionSettings inherit='None'> 
            <connectToConsole>True</connectToConsole> 
            <startProgram /> 
            <port>3389</port> 
            <loadBalanceInfo /> 
        </connectionSettings> 
        </server> 
'@ 
    Add-content $fullpath $serverData 
} 
 
Add-content $fullpath @' 
   </group> 
'@ 
} 
Add-content $fullpath @' 
  </file> 
  <connected /> 
  <favorites /> 
  <recentlyUsed /> 
</RDCMan> 
'@ 
Write-host 'Process Complete. RDG file has been output to $fullPath' -ForegroundColor Green 
start $fullPath 




**

Creates an RDG file for Remote Desktop Connection Manager continaing all your Azure ARM Endpoints. Script can use internal IP addresses or External IP Addresses if VM contains External IP.


Script cycles through all subscriptions associated with your logged on account and creates an RDG file for all VMs on all subscriptions that you can connect to
Uses Public IP Address if it exists
Uses Private IP Address in there is no Public IP
if Hosts are behind a Load Balancer on a custom port, RDG file will have to be manually edited where port is not 3389
RDG file will be output to the location specified in the $wd variable (Default: Desktop)



Execute from Powershell or ISE. RDG file is created on your desktop by default






        
    
TechNet gallery is retiring! This script was migrated from TechNet script center to GitHub by Microsoft Azure Automation product group. All the Script Center fields like Rating, RatingCount and DownloadCount have been carried over to Github as-is for the migrated scripts only. Note : The Script Center fields will not be applicable for the new repositories created in Github & hence those fields will not show up for new Github repositories.
