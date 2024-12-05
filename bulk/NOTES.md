# Bulk creation and stateful management of Check Point objects with Terraform

```bash
# work folder
cd $(mktemp -d)

# create main.tf file with provider configuration
cat <<EOF > main.tf
terraform {
  required_providers {
    checkpoint = {
      source = "CheckPointSW/checkpoint"
      version = "2.8.1"
    }
  }
}
EOF

# credentials as environment variables
export CHECKPOINT_SERVER="20.229.217.158"
export CHECKPOINT_API_KEY="minFwmFgWYlnvafrdEHrFQ=="

cat main.tf

```

Lets create source of truth for Check Point objects in CSV file.

```bash
# create CSV file with Check Point objects
cat <<EOF > objects.csv
name,ipv4_address,color
pankrac,10.0.0.10,red
servac,192.168.10.10,orange
bonifac,172.16.0.1,blue
EOF

cat objects.csv
```

We make TF code to load CSV file and create host resources in Check Point Management.

```bash
# create hosts.tf file with Terraform code
cat <<EOF > hosts.tf
locals {
  objects = csvdecode(file("objects.csv"))
}

resource "checkpoint_management_host" "csv" {
  for_each = { for obj in local.objects : obj.name => obj }

  name        = each.value.name
  ipv4_address = each.value.ipv4_address
  color       = each.value.color
  tags       = ["terraform", "bulk"]
}

resource "checkpoint_management_publish" "publish" {
  triggers = [timestamp()]
  depends_on = [checkpoint_management_host.csv]
  run_publish_on_destroy = true
}
EOF

terraform fmt
cat hosts.tf
```

Time to initialize Terraform and apply the configuration.

```bash
# Initialize Terraform
terraform init
 # plan and apply changes
terraform apply

# our state has one host per CSV line
terraform state list
# see first host
terraform state show 'checkpoint_management_host.csv["pankrac"]'
```



CSV updates drive Terraform changes. We may do some changes and apply them.

```bash
# update CSV file - add new host
cat <<EOF >> objects.csv
joedoe,172.16.0.9,blue
EOF

cat objects.csv

# update servac host - change color
sed -i 's/orange/cyan/' objects.csv
cat objects.csv

# review and apply changes
terraform apply
```

We may also revert changes in CSV file and apply them to Management.

```bash
cat <<EOF > objects.csv
name,ipv4_address,color
pankrac,10.0.0.10,red
servac,192.168.10.10,orange
bonifac,172.16.0.1,blue
EOF

cat objects.csv

# review and apply changes
terraform apply

# final state
terraform state list
```


We may do final cleanup, remove all objects managed by Terraform.
It will also publish changes to Check Point Management.

```bash
# destroy all objects
terraform destroy
```