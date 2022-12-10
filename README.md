# Project-18
AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 3 – REFACTORING

### Introducing Backend on S3
Each Terraform configuration can specify a backend, which defines where and how operations are performed, where state snapshots are stored, etc.
Take a peek into what the states file looks like. It is basically where terraform stores all the state of the infrastructure in json format.

So far, we have been using the default backend, which is the local backend – it requires no configuration, and the states file is stored locally. This mode can be suitable for learning purposes, but it is not a robust solution, so it is better to store it in some more reliable and durable storage.

The second problem with storing this file locally is that, in a team of multiple DevOps engineers, other engineers will not have access to a state file stored locally on your computer.

To solve this, we will need to configure a backend where the state file can be accessed remotely other DevOps team members. There are plenty of different standard backends supported by Terraform that you can choose from. Since we are already using AWS – we can choose an S3 bucket as a backend.

Another useful option that is supported by S3 backend is State Locking – it is used to lock your state for all operations that could write state. This prevents others from acquiring the lock and potentially corrupting your state. State Locking feature for S3 backend is optional and requires another AWS service – DynamoDB


Please note below 
You must be aware that Terraform stores secret data inside the state files. Passwords, and secret keys processed by resources are always stored in there. Hence, you must consider to always enable encryption. You can see how we achieved that with server_side_encryption_configuration.

In previous projects, locks were handled with a local file as shown in terraform.tfstate.lock.info. Since we now have a team mindset, causing us to configure S3 as our backend to store state file, we will do the same to handle locking. Therefore, with a cloud storage database like DynamoDB, anyone running Terraform against the same infrastructure can use a central location to control a situation where Terraform is running at the same time from multiple different people.
Dynamo DB resource for locking and consistency checking:

- With help of remote backend and locking configuration that we have just configured, collaboration is no longer a problem.

- However, there is still one more problem: Isolation Of Environments. Most likely we will need to create resources for different environments, such as: Dev, sit, uat, preprod, prod, etc.

This separation of environments can be achieved using one of two methods:

a. Terraform Workspaces
b. Directory based separation using terraform.tfvars file

WHEN TO USE WORKSPACES OR DIRECTORY?
To separate environments with significant configuration differences, use a directory structure. Use workspaces for environments that do not greatly deviate from each other, to avoid duplication of your configurations. Try both methods in the sections below to help you understand which will serve your infrastructure best.

For now, you can read more about both alternatives here and try both methods yourself, but we will explore them better in next projects.

Security Groups refactoring with dynamic block
For repetitive blocks of code you can use dynamic blocks in Terraform, to get to know more how to use them – watch this video.

Refactor Security Groups creation with dynamic blocks.

EC2 refactoring with Map and Lookup
Remember, every piece of work you do, always try to make it dynamic to accommodate future changes. Amazon Machine Image (AMI) is a regional service which means it is only available in the region it was created. But what if we change the region later, and want to dynamically pick up AMI IDs based on the available AMIs in that region? This is where we will introduce Map and Lookup functions.

Map uses a key and value pairs as a data structure that can be set as a default type for variables.

variable "images" {
    type = "map"
    default = {
        us-east-1 = "image-1234"
        us-west-2 = "image-23834"
    }
}
To select an appropriate AMI per region, we will use a lookup function which has following syntax: lookup(map, key, [default]).

Note: A default value is better to be used to avoid failure whenever the map data has no key.

resource "aws_instace" "web" {
    ami  = "${lookup(var.images, var.region), "ami-12323"}
}
Now, the lookup function will load the variable images using the first parameter. But it also needs to know which of the key-value pairs to use. That is where the second parameter comes in. The key us-east-1 could be specified, but then we will not be doing anything dynamic there, but if we specify the variable for region, it simply resolves to one of the keys. That is why we have used var.region in the second parameter.

Conditional Expressions
If you want to make some decision and choose some resource based on a condition – you shall use Terraform Conditional Expressions.

In general, the syntax is as following: condition ? true_val : false_val

Read following snippet of code and try to understand what it means:

resource "aws_db_instance" "read_replica" {
  count               = var.create_read_replica == true ? 1 : 0
  replicate_source_db = aws_db_instance.this.id
}
true #condition equals to ‘if true’
? #means, set to ‘1`
: #means, otherwise, set to ‘0’

Terraform Modules and best practices to structure your .tf codes
By this time, you might have realized how difficult is to navigate through all the Terraform blocks if they are all written in a single long .tf file. As a DevOps engineer, you must produce reusable and comprehensive IaC code structure, and one of the tool that Terraform provides out of the box is Modules.

Modules serve as containers that allow to logically group Terraform codes for similar resources in the same domain (e.g., Compute, Networking, AMI, etc.). One root module can call other child modules and insert their configurations when applying Terraform config. This concept makes your code structure neater, and it allows different team members to work on different parts of configuration at the same time.

You can also create and publish your modules to Terraform Registry for others to use and use someone’s modules in your projects.

Module is just a collection of .tf and/or .tf.json files in a directory.

You can refer to existing child modules from your root module by specifying them as a source, like this:

module "network" {
  source = "./modules/network"
}
Note that the path to ‘network’ module is set as relative to your working directory.

Or you can also directly access resource outputs from the modules, like this:

resource "aws_elb" "example" {
  # ...

  instances = module.servers.instance_ids
}
In the example above, you will have to have module ‘servers’ to have output file to expose variables for this resource.