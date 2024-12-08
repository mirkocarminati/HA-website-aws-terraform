# HA-website-aws-terraform
Deploying a highly available website on AWS using Terraform

## Introduction

At the beginning of the project, we have a partial Terraform configuraiton (main.tf, variables.tf, outputs.tf) describing a VPC, two private subnets and two instances running the Apache web server.

Our goal is to create a highly available configuration for this website:

- Configure existing instances to serve a custom website
- Create an Elastic Load Balancer with the instances in its pool
- Create public subnets and security groups to secure the deployment

We can run this command to list the resources currently managed by Terraform:
  
```
$ terraform state list
```

## Configuring the Website on the EC2 Instances

Let's replace the instance configuration block with a new one that serves a custom website (a simple website that echoes back the instance's ID) through a user data script.

Use the sed command to replace the current instance configuration in the _main.tf_ file:

```
sed -i '/.*aws_instance.*/,$d' main.tf
```
This command deletes all the lines from the file starting from the line matching _aws_instance_. The instance configuration is the last block in the file so all the other configuration is preserved.

We then append another resource block to configure the instances to serve a custom website that echoes back the instance's ID.

```
cat >> main.tf <<'EOF'
```

The configuration uses _user_data_ to bootstrap the instances to serve a custom website. The file built-in interpolation function reads the contents of the file into a string.
The script creates a file called _index.php_ in the default serving directory of the Apache web server. The PHP code gets the instance ID from the instance's metadata and echoes it back to the user.

Let's view the execution plan for the configuration change:

```
terraform plan
```

And apply changes:

```
terraform apply
```

The plan tells you that Terraform must destroy and then create replacement instances. This is because the _user_data_ must be executed when an instance is first launched.


