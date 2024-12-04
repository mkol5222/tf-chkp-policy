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