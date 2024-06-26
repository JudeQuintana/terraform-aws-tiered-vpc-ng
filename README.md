# Tiered VPC-NG
`v1.0.1`
- Private subnets now have a `special` attribute option like the public subnets.
  - Now AZs can have private subnets only, public subnets only or both!
  - Only 1 subnet, private or public can have `special = true` for VPC attachments per AZ when passed to Centralized Router `v1.0.1`.
  - Private route tables are built per AZ

- Public subnets now have a `natgw` attribute insted of having `enable_natgw`.
  - Tag any public subnet with `natgw = true` to build the NATGW for all private subnets within the same AZ.
  - `special` and `natgw` attributes can also be enabled together on a public subnet
  - Only one public route table is built and shared across all public subnets in the VPC.

- IGW now auto toggles if any public subnets are defined.
- Set required `aws` provider version to >=5.31 and Terraform version >=1.4
- Update deprecated `aws_eip` attribute
- Not ideal to migrate to `v1.0.1` since there are many resource naming changes.
  - Didnt have time figure out a move block migration.
  - Recommend starting fresh at `v1.0.1`

`v1.0.1` example:
```
locals {
  tiered_vpcs = [
    {
      name         = "app"
      network_cidr = "10.0.0.0/20"
      azs = {
        a = {
          public_subnets = [
            { name = "random1", cidr = "10.0.3.0/28" },
            { name = "haproxy1", cidr = "10.0.4.0/26" },
            { name = "other", cidr = "10.0.10.0/28", special = true }
          ]
        }
        b = {
          private_subnets = [
            { name = "cluster2", cidr = "10.0.1.0/24" },
            { name = "random2", cidr = "10.0.5.0/24", special = true }
          ]
        }
      }
    },
    {
      name         = "cicd"
      network_cidr = "172.16.0.0/20"
      azs = {
        b = {
          private_subnets = [
            { name = "jenkins1", cidr = "172.16.5.0/24" }
          ]
          public_subnets = [
            { name = "other", cidr = "172.16.8.0/28", special = true },
            { name = "natgw", cidr = "172.16.8.16/28", natgw = true }
          ]
        }
      }
    },
    {
      name         = "general"
      network_cidr = "192.168.0.0/20"
      azs = {
        c = {
          private_subnets = [
            { name = "db1", cidr = "192.168.10.0/24", special = true }
          ]
        }
      }
    }
  ]
}

module "vpcs" {
  source  = "JudeQuintana/tiered-vpc-ng/aws"
  version = "1.0.1"

  for_each = { for t in local.tiered_vpcs : t.name => t }

  env_prefix       = var.env_prefix
  region_az_labels = var.region_az_labels
  tiered_vpc       = each.value
}
```

`v1.0.0`
- Create VPC tiers
  - Thinking about VPCs as tiers helps narrow down the context (ie app, db, general) and maximize the use of smaller network size when needed.
  - You can still have 3 tiered networking (ie lbs, dbs for an app) internally to the VPC.
- VPC resources are IPv4 only.
  - No IPv6 configuration for now.

- Creates structured VPC resource naming and tagging.
  - `<env_prefix>-<tier_name>-<public|private>-<region|az>`

- An Intra VPC Security Group is created by default.
 - This will be for adding security group rules that are inbound only for access across VPCs.

- Can access subnet id by subnet name map via output.
- Can rename public and private subnets directly without forcing a new subnet resource.
- Public subnets now have a special attribute option.
  - Only one can have `special = true`
  - Associate a NAT Gateway if `enable_natgw = true`.
    - Use for associating VPC attatchments when Tiered VPC is passed to a Centralized Router (one in each AZ).
     - Existing public subnets can be rearranged in any order in their repective subnet list without forcing new resources.
  - The trade off is always having to allocate one public subnet per AZ, even if you don’t need to use it (ie using private subnets only).
- Important:
  - All VPC names should be unique across regions (validation enforced).
  - I highly recommend allocating a small public subnet like a /28 and setting it’s special attribute to true for each AZ.
  - Subnets can only be deleted when they are not in use (ie natgws, vpc attachment, ec2 instances, or some other aws server).

- Recommendations:
  - all vpc names and network cidrs should be unique across regions
  - the routers will enforce uniqueness along with other validations

`v1.0.0` example:
```
locals {
  tiered_vpcs = [
    {
      name         = "app"
      network_cidr = "10.0.0.0/20"
      azs = {
        a = {
          private_subnets = [
            { name = "cluster1", cidr = "10.0.0.0/24" }
          ]
          public_subnets = [
            { name = "random1", cidr = "10.0.3.0/28" },
            { name = "haproxy1", cidr = "10.0.4.0/26" },
            { name = "natgw", cidr = "10.0.10.0/28", special = true }
          ]
        }
        b = {
          private_subnets = [
            { name = "cluster2", cidr = "10.0.1.0/24" },
            { name = "random2", cidr = "10.0.5.0/24" }
          ]
          public_subnets = [
            { name = "random3", cidr = "10.0.6.0/24", special = true}
          ]
        }
      }
    },
    {
      name         = "cicd"
      network_cidr = "172.16.0.0/20"
      azs = {
        b = {
          enable_natgw = true
          private_subnets = [
            { name = "jenkins1", cidr = "172.16.5.0/24" }
          ]
          public_subnets = [
            { name = "natgw", cidr = "172.16.8.0/28", special = true }
          ]
        }
      }
    },
    {
      name         = "general"
      network_cidr = "192.168.0.0/20"
      azs = {
        c = {
          private_subnets = [
            { name = "db1", cidr = "192.168.10.0/24" }
          ]
          public_subnets = [
            { name = "random1", cidr = "192.168.13.0/28", special = true }
          ]
        }
      }
    }
  ]
}

module "vpcs" {
  source  = "JudeQuintana/tiered-vpc-ng/aws"
  version = "1.0.0"

  for_each = { for t in local.tiered_vpcs : t.name => t }

  env_prefix       = var.env_prefix
  region_az_labels = var.region_az_labels
  tiered_vpc       = each.value
}
```

# Networking Trifecta Demo
Blog Post:
[Terraform Networking Trifecta ](https://jq1.io/posts/tnt/)

Main:
- [Networking Trifecta Demo](https://github.com/JudeQuintana/terraform-main/tree/main/networking_trifecta_demo)
  - See [Trifecta Demo Time](https://jq1.io/posts/tnt/#trifecta-demo-time) for instructions.

## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >=1.4 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >=5.31 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >=5.31 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [aws_eip.this_public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip) | resource |
| [aws_internet_gateway.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway) | resource |
| [aws_nat_gateway.this_public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway) | resource |
| [aws_route.this_private_route_out](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route.this_public_route_out](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) | resource |
| [aws_route_table.this_private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table) | resource |
| [aws_route_table.this_public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table) | resource |
| [aws_route_table_association.this_private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association) | resource |
| [aws_route_table_association.this_public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association) | resource |
| [aws_security_group.this_intra_vpc](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) | resource |
| [aws_subnet.this_private](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet) | resource |
| [aws_subnet.this_public](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet) | resource |
| [aws_vpc.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc) | resource |
| [aws_caller_identity.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity) | data source |
| [aws_region.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/region) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_env_prefix"></a> [env\_prefix](#input\_env\_prefix) | prod, stage, test | `string` | n/a | yes |
| <a name="input_region_az_labels"></a> [region\_az\_labels](#input\_region\_az\_labels) | Region and AZ names mapped to short naming conventions for labeling | `map(string)` | n/a | yes |
| <a name="input_tags"></a> [tags](#input\_tags) | Additional Tags | `map(string)` | `{}` | no |
| <a name="input_tiered_vpc"></a> [tiered\_vpc](#input\_tiered\_vpc) | Tiered VPC configuration | <pre>object({<br>    name         = string<br>    network_cidr = string<br>    azs = map(object({<br>      private_subnets = optional(list(object({<br>        name    = string<br>        cidr    = string<br>        special = optional(bool, false)<br>      })), [])<br>      public_subnets = optional(list(object({<br>        name    = string<br>        cidr    = string<br>        special = optional(bool, false)<br>        natgw   = optional(bool, false)<br>      })), [])<br>    }))<br>    enable_dns_support   = optional(bool, true)<br>    enable_dns_hostnames = optional(bool, true)<br>  })</pre> | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_account_id"></a> [account\_id](#output\_account\_id) | n/a |
| <a name="output_default_security_group_id"></a> [default\_security\_group\_id](#output\_default\_security\_group\_id) | n/a |
| <a name="output_full_name"></a> [full\_name](#output\_full\_name) | n/a |
| <a name="output_id"></a> [id](#output\_id) | n/a |
| <a name="output_intra_vpc_security_group_id"></a> [intra\_vpc\_security\_group\_id](#output\_intra\_vpc\_security\_group\_id) | n/a |
| <a name="output_name"></a> [name](#output\_name) | n/a |
| <a name="output_network_cidr"></a> [network\_cidr](#output\_network\_cidr) | n/a |
| <a name="output_private_route_table_ids"></a> [private\_route\_table\_ids](#output\_private\_route\_table\_ids) | n/a |
| <a name="output_private_special_subnet_ids"></a> [private\_special\_subnet\_ids](#output\_private\_special\_subnet\_ids) | n/a |
| <a name="output_private_subnet_cidrs"></a> [private\_subnet\_cidrs](#output\_private\_subnet\_cidrs) | n/a |
| <a name="output_private_subnet_name_to_subnet_id"></a> [private\_subnet\_name\_to\_subnet\_id](#output\_private\_subnet\_name\_to\_subnet\_id) | n/a |
| <a name="output_public_route_table_ids"></a> [public\_route\_table\_ids](#output\_public\_route\_table\_ids) | n/a |
| <a name="output_public_special_subnet_ids"></a> [public\_special\_subnet\_ids](#output\_public\_special\_subnet\_ids) | n/a |
| <a name="output_public_subnet_cidrs"></a> [public\_subnet\_cidrs](#output\_public\_subnet\_cidrs) | n/a |
| <a name="output_public_subnet_name_to_subnet_id"></a> [public\_subnet\_name\_to\_subnet\_id](#output\_public\_subnet\_name\_to\_subnet\_id) | n/a |
| <a name="output_region"></a> [region](#output\_region) | n/a |
