# HA-website-aws-terraform
Deploying a highly available website on AWS using Terraform.

## Introduction

At the beginning of the project, we have a partial Terraform configuraiton (_main.tf_, _variables.tf_, _outputs.tf_) describing a VPC, two private subnets and two instances running the Apache web server.

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

The plan tells us that Terraform must destroy and then create replacement instances. This is because the _user_data_ must be executed when an instance is first launched.


## Configuring Network Resources for Elastic Load Balancing 

We now have two instances hosting the custom website. The instances are running in private subnets in different availability zones and are assigned to the default security group for the VPC that allows all traffic. There are a few resources that need to be created to allow for an Elastic Load Balancer (ELB) to securely distribute traffic between the instances:

- Public subnets for each availability zone must be created so the load balancer can be accessed from the internet. This requires additional resources such as an internet gateway to connect to the internet, and route tables that route to the internet.
- A security group to allow traffic from the internet to the public subnets that will house the ELB on port 80 (HTTP)
- A security group to allow traffic from the ELB in the public subnets to the instances in the private subnets on port 80 (HTTP)

These networking resouces are defined in the _networking.tf_ file.

Next, we define security groups that will secure traffic into the public and private subnets in a configuration file called _security.tf_.
We also need to update the _main.tf_ file to attach the web server security group.

After these changes, let's run _terraform apply_:

```
terraform apply
```

This change doesn't require recreation and an update in-place can be performed.

## Configuring an Elastic Load Balancer

The final step is to create an ELB so that the website will be highly-available. In case one instance is down, the ELB will continue serving the website from the other instance.
The ELB is defined in the _load_balancer.tf_ file. It is internet-facing by default.

We will add an output that will save the DNS address of the ELB as the website address.

Let's apply the configuration changes: 

```
terraform apply
```

Now let's store the site_address output in a shell variable and remove the quotation marks.

```
site_address=$(terraform output site_address)
site_address=${site_address//\"/}
```

To verify the ELB is working, send an HTTP request to the ELB every two seconds using the watch and curl command:

```
watch curl -s $site_address
```

It takes about a minute for the ELB to start sending requests to the instances. We will eventually see messages from the web server instances and notice the instance ID changing between two values. This verifies the ELB is distributing the traffic to all of the instances. Press _ctrl+c_ to stop the watch command.

