## Full Jenkins installation via Terraform in AWS.
#### This makes Jenkins portable and reusable :)

This is a full installation of Jenkins including a backup job via Terraform in AWS. Once installed, configured and backed up you can restore your jenkins instance using the same method as a new install - restoring all your backed up files.

#### Configuration components:
**files/plugins.txt** - this file is used to populate the desired plugins to install in Jenkins. If you're installing for the first time and don't know which plugins you need use the pre-populated list of plugins provided in the repo. If you have an existing instance and you want the same plugins installed you can run the following command to extract a list of plugins from your existing Jenkins instance:
```Shell
curl -s -k 'http://<adminUserName>:<adminPassWord>@localhost:8080/pluginManager/api/json?depth=1' \ | jq -r '.plugins[].shortName' | tee plugins.txt
```
but make sure you have jq installed first:
```Shell
sudo yum install jq -y
```
and then copy the `plugins.txt` file to the `files` subdirectory overwriting the current `plugins.txt` file.

**files/terraform.tfvars** - this file is included in your `.gitignore`. When executing terraform you'll use this file to populate the required variabled needed for Terraform to run. Create and populate the following:
- access_key = "Your AWS access key"
- secret_key = "Your AWS secret key"
- region = "Regiong to deploy Jenkins to"
- zone_id = "If you're using the Route 53 module the zone to create the record in"
- record_name = "The desired Route 53 record name"
- bucket_id = "S3 bucket to store Jenkins backup and config files in"
- key_name = "keypair name for EC2 SSH access You must create the keypair in AWS manually. This does not get created through Terraform"

**files/jenkins.env** - this files is included in your `.gitignore`. This file is copied to S3 durring Terraform run and is used to generate the initial admin username and password for Jenkins via `files/init.groovy`. Create and populate the following:
- admin_username=adminUserName
- admin_password=adminPassword

#### Deploying Jenkins
Once you have configured the above you're ready to deploy Jenkins! We're going to assume you already have Terraform installed and configured. If not please follow [this link](https://learn.hashicorp.com/terraform/getting-started/install.html) to get Terraform setup.

Open terminal and change to the repo directory > prod (Jenkins/prod) then type:
```Shell
terraform init
```
to initialize Terraform and the associated modules. Next, let's apply our proposed changes to deploy Jenkins:
```Shell
terraform apply --var-file=../files/terraform.tfvars
```
This will generate a plan of what will be applied during the run. Type `yes` to approve and continue:

![apply_plan.png](/readmeFiles/apply_plan.png)

Here's a breakdown of what's being installed on AWS by module (`Jenkins/modules`) and sorted by order:
- S3 - Copies the neccessary files to an pre-created S3 bucket (see `files/terraform.tfvars`) for configuring Jenkins installation. Below is a list of files and description:
    - `init.groovy` - startup script to create initial Jenkins admin account and disable annon UI read access.
    - `install-plugins.sh` - script used to install plugins and dependencies from `plugins.txt`. **Ripped off from https://gist.github.com/micw/e80d739c6099078ce0f3**.
    - `plugins.txt` - list of plugins to install. To extract a list of plugins from an existing Jenkins instance run the [extraction command](#configuration-components) on the Jenkins server
    - `jobs.tgz` - contains our pre-baked backup job.

- VPC - The VPC for Jenkins to run in including subnets, route tables, ACL, routes and associations.
- SEC - Security groups for EC2, ELB and role needed for EC2 to read/write from S3.
- EC2 - Launch config, autoscaling group, ELB and associations.
- Route53 - www record for Jenkins instance.

#### Post install steps
1. Log into Jenkins using the username and password provided in `jenkins.env`.

2. Configure the Jenkins backup job and include the S3 bucket used in your Terraform config (`Jenkins/files/terraform.tfvars`) in the paramter `S3_BUCKET`: ![config_param.png](/readmeFiles/config_param.png) From here you can also configure the job to run on a schedule if desired: ![config_schedule.png](/readmeFiles/config_schedule.png)

3. Run the Jenkins backup job and cross your fingers :) Just kidding. It SHOULD work.

### If you have any issues with any part of the process or questions feel free to [open an issue in Github](https://github.com/bspeagle/Jenkins/issues) and I'll respond as soon as I can. Thanks for trying it out!