# StorageGRID Webscale PowerShell Cmdlet Tutorial

This tutorial will give an introduction to the StorageGRID Webscale and S3 PowerShell Cmdlets.

## Discovering the available Cmdlets

Since PowerShell 3 all available Modules are automatically loaded. To explicitly load the StorageGRID-Webscale Module use

```powershell
Import-Module -Name StorageGRID-Webscale
```

List all Cmdlets included in the StorageGRID-Webscale Module

```powershell
Get-Command -Module StorageGRID-Webscale
```

List all Cmdlets included in the S3 Client (this command only works with PowerShell 6 and later)

```powershell
Get-Command -Module S3-Client
```

Show the syntax of all Cmdlets from the StorageGRID-Webscale Module

```powershell
Get-Command -Module StorageGRID-Webscale -Syntax
```

Show the syntax of all Cmdlets from the StorageGRID-Webscale Module (this command only works with PowerShell 6 and later)

```powershell
Get-Command -Module S3-Client -Syntax
```

To get detailed help including examples for a specific Cmdlet (e.g. for Connect-SgwServer) run

```powershell
Get-Help Connect-SgwServer -Detailed
```

## Profile Management

It is recommended to use profiles to manage the connections to StorageGRID. The profile contains the login information which will be saved under

```powershell
Add-SgwProfile -ProfileName MyProfile -Name adminnode.example.com -Credential (Get-Credential)
```

## Tutorial for Tenant Users

This section is dedicated to Tenant Users. If you are a Grid Administrator, check section [Tutorial for Grid Administrators]()

## Connect to a StorageGRID Management Server

For data retrieval a connection to the StorageGRID Management Server (also referred to as "Admin Node") is required. The `Connect-SgwServer` Cmdlet expects the hostname or IP (including port if different than 80 for HTTP or 443 for HTTPS) and the credentials for authentication

```powershell
$Name = "admin-node.example.org"
$Credential = Get-Credential
$Server = Connect-SgwServer -Name $Name -Credential $Credential
```

If the login fails, it is often due to an untrusted certificate of the StorageGRID Management Server. You can ignore the certificate check with the `-SkipCertificateCheck` option

```powershell
$Server = Connect-SgwServer -Name $Name -Credential $Credential -SkipCertificateCheck
```

By default the connection to the StorageGRID Webscale Server is established through HTTPS. If that doesn't work, HTTP will be tried.

The Cmdlets allow to connect either as Grid Administrator or as Tenant User. To connect to the StorageGRID Management Server as a tenant user specify the AccountId of the tenant. Ask a Grid Administrator for the AccountId of the tenant if you don't know it.

```powershell
Connect-SgwServer -Name $Name -Credential $Credential -AccountId $AccountId
```

## Tenant Management

To create a new S3 tenant use the following steps.

```powershell
$Credential = Get-Credential -UserName root
$Account = New-SgwAccount -Name "MyTenant" -Capabilities s3,management -Password $Credential.GetNetworkCredential().Password
```

Check that Account was created either by listing all accounts

```powershell
Get-SgwAccounts
```

or by checking for the individual account

```powershell
$Account | Get-SgwAccount
```

Log in with the root user of the new tenant
```powershell
$Account | Connect-SgwServer -Name $Name -Credential $Credential
```

Then create S3 credentials and remember to store the access key and secret access key in a secure location
```powershell
New-SgwS3AccessKey
```

Check that S3 access keys have been created by listing all access key

```powershell
Get-SgwS3AccessKeys
```

or by checking for the individual account

```powershell
$AccessKey | Get-SgwS3AccessKey
```

The tenant can only be removed by a grid administrator. If you have stored the grid admin connection in the variable `$Server` as shown above, then you can use it here

```powershell
$Account | Remove-SgwAccount -Server $Server
```

Otherwise connect as grid administrator again and then run

```powershell
$Account | Remove-SgwAccount
```

## Creating local groups and users

Local groups and users can be used to grant full or restricted access to the resources of a tenant to e.g. a specific application.

Make sure that you are connected to the tenant as root user or user with privileges to manager users and groups (check the previous section for details).

```powershell
$Account | Connect-SgwServer -Name $Name -Credential $Credential
```

If you want to restrict S3 access by denying bucket creation, deletion, policy change and versioning change as well as object version deletion create the following policy

```powershell
$GroupPolicy = New-AwsPolicy -Principal $null
$GroupPolicy = $GroupPolicy | Add-AwsPolicyStatement -Principal $null -Effect Deny -Action s3:CreateBucket,s3:DeleteBucket,s3:DeleteBucketPolicy,s3:PutBucketPolicy,s3:PutBucketVersioning,s3:DeleteObjectVersion
```

Now create a new group with the previously created policy

```powershell
$Group = New-SgwGroup -UniqueName "mygroup" -Type "local" -S3Policy $GroupPolicy
```

Users of this group will not be able to log into the tenant portal. If you want to allow Tenant Portal access specify either of the parameters `-RootAccess`,`-ManageAllContainers`,`-ManageOwnS3Credentials`.

If you want to allow full S3 access use the following to create the group

```powershell
$Group = New-SgwGroup -UniqueName "mygroup" -Type "local" -S3FullAccess
```

Now create a local user

```powershell
$UserCredential = Get-Credential -UserName "myuser" -Message "Insert the password for the user"
$User = $Group | New-SgwUser -UniqueName "myuser" -Password $UserCredential.GetNetworkCredential().Password
```

## Export account usage to CSV

As a tenant user, the Account usage can be retrieved with

```powershell
Get-SgwAccountUsage
```

A grid administrator can use this simple workflow to retrieve the S3 account usage data and export it as CSV to [$HOME\Downloads\TenantAccounting.csv]($HOME\Downloads\TenantAccounting.csv).

```powershell
$TenantAccounting = foreach ($Account in (Get-SgwAccounts | Where-Object { $_.capabilities -match "s3" })) {
    $Usage = $Account | Get-SgwAccountUsage
    $Output = New-Object -TypeName PSCustomObject -Property @{Name=$Account.name;ID=$Account.id;"Calculation Time"=$Usage.calculationTime;"Object Count"=$Usage.objectCount;"Data Bytes used"=$Usage.dataBytes}
    Write-Output $Output
}

$TenantAccounting | Export-Csv -Path $HOME\Downloads\TenantAccounting.csv -NoTypeInformation
```

The next workflow will retrieve the S3 Account usage per Bucket and export it as CSV to [$HOME\Downloads\BucketAccounting.csv]($HOME\Downloads\BucketAccounting.csv)

```powershell
$BucketAccounting = foreach ($Account in (Get-SgwAccounts | Where-Object { $_.capabilities -match "s3" })) {
    $Usage = $Account | Get-SgwAccountUsage
	foreach ($Bucket in $Usage.Buckets) {
		$Output = New-Object -TypeName PSCustomObject -Property @{Name=$Account.name;ID=$Account.id;"Calculation Time"=$Usage.calculationTime;Bucket=$Bucket.Name;"Object Count"=$Bucket.objectCount;"Data Bytes used"=$Bucket.dataBytes}
		Write-Output $Output
	}
}

$BucketAccounting | Export-Csv -Path $HOME\Downloads\BucketAccounting.csv -NoTypeInformation
```

## Experimental Accounting of disk usage per tenant and bucket

The following script expects that the S3-Client PowerShell Module is installed and that a connection to a server was established via `Connect-SgwServer` and the minimum API Version was set to 1 with `Update-SgwConfigManagement -MinApiVersion 1`.

```powershell
$Accounts = Get-SgwAccounts -Capabilities s3
$TenantAccounting = @()
$BucketAccounting = @()
foreach ($Account in $Accounts) {
    Write-Host "Checking account $($Account.Name)"
    $AccountUsage = 0
    $AccountObjectCount = 0
    $Buckets = $Account | Get-S3Buckets
    foreach ($Bucket in $Buckets) {
        Write-Host "Checking bucket $($Bucket.BucketName)"
        $BucketUsage = 0
        $BucketObjectCount = 0
        if ($Bucket) {
            $Objects = $Bucket | Get-S3Objects
            foreach ($Object in $Objects) {
                $BucketObjectCount += 1
                $ObjectMetadata = $Object | Get-SgwObjectMetadata
                $ReplicaCount = ($ObjectMetadata.locations | ? { $_.type -eq "replicated" }).Count
                $ErasureCodeDataCount = ($ObjectMetadata.locations.fragments | ? { $_.type -eq "data" }).Count
                $ErasureCodeParityCount = ($ObjectMetadata.locations.fragments | ? { $_.type -eq "parity" }).Count
                if ($ReplicaCount) {
                    $SizeFactor = $ReplicaCount
                }
                elseif ($ErasureCodeDataCount) {
                    $SizeFactor = ($ErasureCodeDataCount + $ErasureCodeParityCount) / $ErasureCodeDataCount
                }
                elseif ($ObjectMetadata.objectSizeBytes -gt 0) {
                    Write-Warning "No replicas and EC parts found for object ${Bucket.BucketName}/$($Object.Key)"
                }
                $SizeFactor = $ReplicaCount
                if ($ObjectMetadata.diskSizeBytes) {
                    $BucketUsage += $SizeFactor * $ObjectMetadata.diskSizeBytes
                }
                else {
                    $BucketUsage += $SizeFactor * $ObjectMetadata.objectSizeBytes
                }
            }
            $BucketAccounting += [PSCustomObject]@{AccountName=$Account.Name;AccountId=$Account.Id;BucketName=$Bucket.BucketName;BucketUsage=$BucketUsage;BucketObjectCount=$BucketObjectCount}
            $AccountUsage += $BucketUsage
            $AccountObjectCount += $BucketObjectCount
        }
    }
    $TenantAccounting += [PSCustomObject]@{AccountName=$Account.Name;AccountId=$Account.Id;AccountUsage=$AccountUsage;AccountObjectCount=$AccountObjectCount}
}
$BucketAccounting | Export-Csv -Path $HOME\Downloads\BucketAccounting.csv -NoTypeInformation
$TenantAccounting | Export-Csv -Path $HOME\Downloads\TenantAccounting.csv -NoTypeInformation
```

## Report creation

StorageGRID allows grid administrators to create reports for different metrics on grid, site and node level.

You can check the available metrics by using Command Completion. Just type the following command and hit the Tabulator key (->|)

```powershell
Get-SgwReport -Attribute
```

The reports have a start and an end date. By default the attributes will be listed for the last hour. For attributes within the last 7 days StorageGRID will report the measure value in a specific interval. Attributes older than 7 days will be aggregated and StorageGRID will return average, minium and maximum value for each sample time. You can specify a time range using PowerShell DateTime objects.

Here are some examples:

```powershell
Get-SgwReport -Attribute 'S3 Ingest - Rate (XSIR) [MB/s]'
Get-SgwReport -Attribute 'S3 Retrieval - Rate (XSRR) [MB/s]' -StartTime (Get-Date).AddDays(-1)
Get-SgwReport -Attribute 'Used Storage Capacity (XUSC) [Bytes]' -StartTime (Get-Date).AddDays(-10) -EndTime (Get-Date).AddDays(-1)
```

By default the report is created for all nodes in the grid. To select a specific site specify the `-site` parameter

```powershell
Get-SgwReport -Attribute 'S3 Ingest - Rate (XSIR) [MB/s]' -Site $Site
```

To select a specific node specify the `-node` parameter

```powershell
Get-SgwReport -Attribute 'S3 Ingest - Rate (XSIR) [MB/s]' -Node $Node
```

## Retrieving Metrics

Retrieving individual metrics via REST API is possible since StorageGRID 11.0. To list the available metrics, run

```powershell
Get-SgwMetricNames
```

StorageGRID uses [Prometheus](https://prometheus.io/) to collect the metrics and [Prometheus Queries](https://prometheus.io/docs/querying/basics/) can be issued via e.g.

```powershell
Get-SgwMetricQuery -Query 'storagegrid_s3_operations_successful'
```