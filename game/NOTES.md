# Game

Lets try Terraform on Windows, in powershell command line. It is just one .exe after all.

Enter powershell command prompt and by Win-R (Run) and executing `powershell` command.

```powershell
# make temporary directory and enter it
New-TemporaryFile | %{ rm $_; mkdir $_; cd $_ }

# download terraform binary
Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform/1.10.1/terraform_1.10.1_windows_amd64.zip" -OutFile "tf.zip"

# extract terraform binary
expand-archive tf.zip -destinationpath .

# verify it works
ls .\terraform.exe
.\terraform.exe -version
```

Now working folder has Terraform and we just add some code to it. 

```powershell
notepad main.tf
```

Put following code to `main.tf` file and read it. Save the file and exit notepad.
Update your label `user:somename` to your own unique name or alias. Will use it for competition scrore board.

```hcl
terraform {
  required_providers {
    checkpoint = {
      source = "CheckPointSW/checkpoint"
      version = "2.8.1"
    }
  }
}

resource "checkpoint_management_host" "game" {
  name = "game_by_someuser"
  ipv4_address = "127.0.0.127"
  ignore_warnings = true # we ignore warnings like same IP on multiple hosts
  tags = ["game", "user:someuser"]
}

resource "checkpoint_management_publish" "policy" {
  depends_on = [checkpoint_management_host.game]
  triggers = ["${timestamp()}"]
}
```

Continue in powershell:
```powershell
# credentials for Check Point Management
$env:CHECKPOINT_SERVER="20.229.217.158"
$env:CHECKPOINT_API_KEY="NotSoSharpUSEyourOWN=="
# format code
.\terraform.exe fmt
gc main.tf
# fetch provider plugin
.\terraform.exe init
# validate code
.\terraform.exe validate
# detect drift between current managed state and code
.\terraform.exe plan
# plan again for final apply. approve and check GUI, if available
.\terraform.exe apply
```

Now we can create score board with Management API:
```powershell
# login with CHECKPOINT_API_KEY to CHECKPOINT_SERVER
$url = "https://$env:CHECKPOINT_SERVER/web_api/login"
$params = @{
    uri = $url
    method = "POST"
    headers = @{"Content-Type"="application/json"}
    body = ConvertTo-Json @{ "api-key" = $env:CHECKPOINT_API_KEY }
}
# powershell 5 certificate valuidation bypass
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true} ;
# powershell version
$PSVersionTable
# login
$response = Invoke-RestMethod @params
# or in Powershell 7
$response = Invoke-RestMethod @params -SkipCertificateCheck
# get session token
$sid = $response.sid
Write-Host $sid

# fetch host objects with tag game
# https://sc1.checkpoint.com/documents/latest/APIs/#web/show-hosts~v2%20

$url = "https://$env:CHECKPOINT_SERVER/web_api/show-hosts"
$params = @{
    uri = $url
    method = "POST"
    headers = @{"Content-Type"="application/json"; "X-chkp-sid"=$sid}
    body = ConvertTo-Json @{ "filter" = "game"; "details-level" = "full" ; "limit" = 100 }
}
# fetch hosts
$response = Invoke-RestMethod @params
# or in Powershell 7
$response = Invoke-RestMethod @params -SkipCertificateCheck

# print hosts
$response.objects | select name, ipv4_address, tags

# one host in detail
$response.objects[0] | ConvertTo-Json -Depth 10

# score board sorted by creation time
$response.objects | select name, ipv4_address, @{n="tags";e={($_.tags | %{$_.name} )}}, @{n="ts"; e={ $_."meta-info"."creation-time".posix }}, @{n="created"; e={ $_."meta-info"."creation-time"."iso-8601" }}| sort-object ts | ft -AutoSize
```