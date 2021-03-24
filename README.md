# Minecraft Server Bootstrap in GCP
Welcome to my small, but humble Terraform module for a vanilla Minecraft server, hosted in [Google Cloud Platform](https://console.cloud.google.com/)!

![meme](https://gifimage.net/wp-content/uploads/2017/08/its-alive-gif-20.gif)

## Table Of Contents
- [Overview](#Overview)
- [How to Use](#How-to-Use)
- [Terraform Configuration](#Terraform-Configuration)
- [General Server Management](#General-Server-Management)
- [Future Goals](#Future-Goals)
- [Inspiration](#Inspiration)

## Overview

This project helps streamlines a majority of the steps needed to take when creating a new multiplayer Minecraft Server. Below describes what this Terraform Module creates:
- One GCE instance
    - Assigned the default service account that is scoped to Read Only for Compute API and Read Write to Cloud Storage.
    - Provided metadata scripts for bootstrapping and shutting down.
    - Provided SSH keys to allow access to one outside user
- One persistent SSD to store the Minecraft Server Data
- One Static IP
- A custom Network with one Subnetwork
- Firewaall rules in the network to only allow specific traffic to:
    - 22
    - 25565
    - icmp
- A Cloud Storage Bucket

The overall cost to run this project varies greatly with usage and instance size, but it should be fairly mimimum if using the project defaults (~$1 per day).

## How to Use

### Requirements

| Name | Version |
|------|---------|
| terraform | > 0.12 |
| google | = 3.60.0 |
| random | = 3.1.0 |
| template | = 2.2.0 |

### Providers

| Name | Version |
|------|---------|
| google | = 3.60.0 |
| random | = 3.1.0 |
| template | = 2.2.0 |

### Pre-Reqs

1. A Google Account (GCP has a [free credit](https://cloud.google.com/free/) sytem where all acccounts can get $300 worth of usage)
2. `terraform` CLI tool, whicch can be gotten [here](https://www.terraform.io/downloads.html)
3. Some light knowledge of Linux, GCP and Terraform
4. Some familiarity with Minecraft server hosting

### Steps

1. Sign into your Google account and log into [GCP](https://console.cloud.google.com/)
2. Create a new __project__ and remember the `project_id` that GCP assigns it.
3. In that new __project__, create a new __service account__ (`IAM & Admin -> Service Accounts`)
    - Name the account to whatever you desire
    - For the __Role__, specify `Owner`
    - Once done, navigate to that new __service account__ and select `Keys -> Create New Key -> JSON`
    - Save this key in a secure location on your computer (I recommend `~/.ssh`).

> Be careful with that key! __It has admin API access to your entire GCP account__, meaning anything can be deployed in GCP using said key. More savy GCP users can use a role that is less wide in scope for this project, but for the sake of this walkthrough, you can proceed with these permissions.

4. Enable the following APIs in the GCP Console:
    - `Compute Engine API` 
5. Create a Terraform directoty, using this [example](./example) as a basis. Make sure to keep these in mind:
    - Change the __project name__ in `main.tf` if you are not using the project name in there.
    - Create a `terraform.tfvars` and define the following values:
        - `creds_json`
        - `ssh_pub_key_file`
        - `game_whitelist_ips`
        - `admin_whitelist_ips`
    - (Optional) Configure the initial settings of the `server.properties` in the `server_properties.tpl`
6. Run `terraform init`
7. Run `terraform plan` (should get 10 new resources created) and it it looks good, `terraform apply`
8. Sit back for a few mins and your new Minecraft Server should be running at the `ip_address` the `terraform apply` outputs!

## Terraform Configuration

### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| admin\_whitelist\_ips | The IPs to allow for SSH and ping access, generally reseved for operational work/troubleshooting. If existing\_subnetwork\_name is specified, this will be ignored. | `list(string)` | n/a | yes |
| backup\_length | How many days will a backup last in the bucket? | `number` | `5` | no |
| creds\_json | The absolute path to the credential file to auth to GCP. This needs to be associated with the GCP project that is being used | `string` | n/a | yes |
| disk\_size | How big do you want the SSD disk to be? Defaults to 50 GB | `string` | `"50"` | no |
| existing\_subnetwork\_name | An existing subnetwork to leverage placing the instances. Assumes that the firewalls in the subnetwork are already configured. | `string` | `""` | no |
| game\_whitelist\_ips | The IPs used to connect to the Minecraft server itself through the MC client. If existing\_subnetwork\_name is specified, this will be ignored. | `list(string)` | n/a | yes |
| machine\_type | The type of machine to spin up. If the instance is struggling, it might be worthwhile to use stronger machines. | `string` | `"n1-standard-2"` | no |
| mc\_home\_folder | The location of the Minecraft server files on the instance | `string` | `"/home/minecraft"` | no |
| mc\_server\_download\_link | The direct download link to download the server jar. Defaults to a link with 1.16.5. | `string` | `"https://launcher.mojang.com/v1/objects/35139deedbd5182953cf1caa23835da59ca3d7cd/server.jar"` | no |
| project\_name | The name of the project. Not to be confused with the project name in GCP; this is moreso a terraform project name. | `string` | `"mc-server-bootstrap"` | no |
| region | The region used to place these resources. Defaults to us-west1 | `string` | `"us-west2"` | no |
| server\_image | The boot image used on the server. Defaults to `ubuntu-1804-bionic-v20191211` | `string` | `"ubuntu-1804-bionic-v20191211"` | no |
| server\_max\_ram | The maximum amount of RAM to allocate to the server process | `string` | `"7G"` | no |
| server\_min\_ram | The minimum amount of RAM to allocate to the server process | `string` | `"1G"` | no |
| server\_property\_template | The file path used to parse the server property file for the MC server. Defaults to the standard one in the module | `string` | `"./templates/server_properties.tpl"` | no |
| ssh\_pub\_key\_file | The SSH public key file to use to connect to the instance as the user specified in ssh\_user | `string` | n/a | yes |
| ssh\_user | The name of the user to allow to SSH into the instance | `string` | `"iamall"` | no |
| zone\_prefix | The zone prefix used for deployments. Defaults to 'a'. | `string` | `"a"` | no |

### Outputs

| Name | Description |
|------|-------------|
| created\_subnetwork | The name of the created subnetwork that was provisioned in this module. Can be used to provision more servers in the same network if desired |
| server\_ip\_address | The public IP address used to access this instance |

### Nuances

While most of the configuration has verbose descriptions, there are some options that have a bit more complexity:

| Terraform Variable     | Notes        |
| :--------------------- | :----------: |
| `machine_type`         | Depending on what you use, this can greatly affect the price of running this. The most cost effective, [N1](https://cloud.google.com/compute/vm-instance-pricing#n1_predefined) should be the one you want to use, unless your server will be hostin a massive amount of players.
| `game_whitelist_ips`           | To ban/allow players into the game, it is handled on the infrastructure level. As such, make sure to get your friend's IPs and place them in here, so they can acces this instance! |
| `admin_whitelist_ips` | This should only be restricted to the person that is administrating this server. Not correlated to `admin` power in Minecraft; this is moreso system admin access |
| `mc_server_download_link` | One easy way to get different versions of Minecraft can be gotten at [this](https://mcversions.net/) link. Just find the right server version, right click on the download URL and save it, placing it in this value. |
| `server_property_template` | This could change consistenty in the server, making it tricky to keep track of in this code. As such, any new changes made after the initial deployment of the server will __NOT__ be reflected in code. To use a new config if one changed it outside of the server, one must manually go onto the instance and edit the config to match what is down in code. |
| `existing_subnetwork_name` | This allows for multiple instances of this module to be deployed in the same network, for easier management. To properly use this, make sure one module of this stack is deployed, with the other module calls referencing the `created_subnetwork` output of the first module. |

## General Server Management

By default, rebooting and respinning up an instance will automatically set up the server for you. No need for any action on you!

Backup, restores and restarts can be performed via the following scripts:
- `/home/minecraft/scripts/backup.sh` (default location)
    - Pushes up current state of server to Cloud Storage Buckets.
    - Ex: `$ cd /home/minecraft/scripts && sudo ./backup.sh`
- `/home/minecraft/scripts/restore_backup.sh` (default location)
    - Restores the server world to the specified state
    - Ex: `$ cd /home/minecraft/scripts && sudo ./restore_backup.sh nameOfBaackup`
- `/home/minecraft/scripts/restart.sh` (default location)
    - Restarts the Minecraft server (not the instance)
    - Ex: `$ cd /home/minecraft/scripts && sudo ./restart.sh`

To keep costs low, it is a good idea to stop this instance when it is not in use. This can be done via the GCP console and/or the CLI.

## Future Goals

- [] Create automated process to perform backups (cron job, ansible)
- [] Create a process to restore backups, possibly allowing the user to see a list of all backups in bucket
- [] Add a curated list of Minecraft versions that allows the end user to just specify the `version` instead of a URL link

## Inspiration

- This [blog](https://cloud.google.com/blog/products/it-ops/brick-by-brick-learn-gcp-by-setting-up-a-minecraft-server) gave me the initial idea on how this can be easily done nowadays
- My current gig involving myself being inmersed in the cloud
- A certain coworker who if you are reading this, you know who exactly :wink: