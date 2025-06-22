- [Ansible and Terraform](#Ansible-and-Terraform)

  - [Intergrate Ansible Playbook execution in Terraform](#Intergrate-Ansible-Playbook-execution-in-Terraform)
 
  - [Wait for EC2 Server to be fully initialized](#Wait-for-EC2-Server-to-be-fully-initialized)
 
  - [Using null resource](#Using-null-resource)

# Ansible-Terraform

## Ansible and Terraform

#### Intergrate Ansible Playbook execution in Terraform

Wrap up: I basically now know how to configure completely new EC2 Instance using Ansible . And for that we frist used Terraform to automatically create EC2 Instance and then we switched back to our Ansible project to execute Ansible Playbook there . 

We had to update the hosts file with the new Server's IP address and then just execute Ansible Playbook command

There is still the link between the provisioning and configuring the server that we have to do manually: 

 - We have to get that IP address from the Terraform output and we have to set that in the hosts file and then execute Ansible command 

I want to automate complete process

 - Provisioning the Server and then basically handing over the control over to Ansbile and then Ansible taking over and configuring the Server without any intervention from our side .

 - Basically I just need to execute `terraform apply`

First I will go to my Terraform project (https://github.com/ManhTrinhNguyen/Terraform-Exercise) 

In this part 

```
resource "aws_instance" "myapp" {
  ami = data.aws_ami.amazon-linux-image.id
  instance_type = var.instance_type
  subnet_id = aws_subnet.myapp-subnet.id 
  vpc_security_group_ids = [aws_security_group.myapp-sg.id]
  availability_zone = var.availability_zone

  associate_public_ip_address = true

  key_name = "terraform-exercise"


  user_data = file("entry-script.sh")

  user_data_replace_on_change = true
  tags = {
    Name = "${var.env_prefix}-myapp"
  }
}
```

I will remove a `user_data`

Inside this provision EC2 Instance block we can add a provisioner that will call Ansible command that will then execute on this instance

This time I will executing the `provisioner` with Ansible command 

Provisioner that we have available : 

 - `local-exec`

 - `remote-exec`

 - `file`
 
In this case I will use `local exec` provisioner

 - The reason why we need `local exec` provisioner bcs we are executing the Ansible command locally on this machine . On the same machine where `terraform apply` command get executed .

 - So the Ansible command that we will specify in `local exec` will execute on our laptop 

<img width="500" alt="Screenshot 2025-05-14 at 14 19 05" src="https://github.com/user-attachments/assets/8aab8dc7-222a-4d0f-aeca-bcd0302fd40b" />

I will execute `ansbile-playbook` command inside `local-exec` like this : `command: "ansible-playbook"`

 - To execute `ansible-playbook` first of all we need a playbook name . We need a path to our Ansible `playbook` : `command = ansible-playbook <path-to-ansible-playbook>`

 - But the path to Ansible playbook very long and not clean and also we don't only have the playbook, but we also don't have the configuration in that folder . We don't want to lose all that configuration when we execute the Playbook, so we want it to be the same as when we execute this playbook . For that I will change to the Ansible project directory before executing the command by using `working_dir= <Path-to-Ansible-Project>`

Another the things we have to do here if we execute this Terraform script with this local exec provisioner, so basically the server will be created with some new IP address and then this Ansible playbook will be executed from the Ansible folder, so which `hosts` will Ansbile conntect to is the `hosts` that is defined in Ansible Directory . 

 - We need someway to dynamically get the IP address instead of getting it from `hosts`

 - So instead of using this `hosts` file in our Ansible playbook command, we actually want to pass the IP address from the terraform itself .

 - And I can do that by setting a flag call `--inventory` . This `--inventory` will be a newly IP address created by Terraform which has to be available a dynamic value and we can actually get that by using `self` object .

 - Within `resource "aws_instance" ""` block the `self` refer to the AWS instance which has an attribute called public IP

 - So when the Server created the public IP address will be retrieved from the `self` object, this will be set as an IP address for the `--inventory ${self.public_ip}`. This should acutally overwrites the hosts file that we have in the Ansbile folder . So the `hosts` file will be ignored so this is not in addition to the hosts file but this is a replacement
 
 - and this `--inventory` flag take a file location as a parameter or a comma separated list of IP addresses. And if I am passing IP address just one IP address in this case, and not a file location I have to do `,` like this : `command = "ansible-playbook --inventory ${self.public_ip}, <Ansible-deploy-file>"` so Ansible knows we are passing the IP address not a file

And now on the IP address, we have to specify the Private key location and the user, just like we did in the `hosts` file 

 - And can specify those two also using flag and also parameterize the value `--private-key ${var.ssh_key_private}` .

 - In the `terraform.tfvars` I will set `ssh_key_private = /User/trinhnguyen/.ssh/id_rsa` . This will be the private key location

 - To specify user `--user ec2-user` . I can also parameterize it as well

   
```
resource "aws_instance" "myapp" {
  ami = data.aws_ami.amazon-linux-image.id
  instance_type = var.instance_type
  subnet_id = aws_subnet.myapp-subnet.id 
  vpc_security_group_ids = [aws_security_group.myapp-sg.id]
  availability_zone = var.availability_zone

  associate_public_ip_address = true

  key_name = "terraform-exercise"


  user_data = file("entry-script.sh")

  user_data_replace_on_change = true
  tags = {
    Name = "${var.env_prefix}-myapp"
  }

  provisioner "local-exec" {
    working_dir= <Path-to-Ansible-Project>
    command = "ansible-playbook --inventory ${self.public_ip}, --private-key ${var.ssh_key_private} --user ec2-user <Ansible-deploy-file>"
  }
}
```

One more thing I have to change in this `Playbook` itself . In `hosts` I am reference `docker_server`. We don't have a Docker Server reference anymore, bcs we are passing in `--inventory` which is just IP address, so we need to reference that IP address instead of reference `docker_server` . I will defind `hosts: all`

#### Wait for EC2 Server to be fully initialized 

So now we have configured executing Ansilbe `Playbook` from Terraform script there is one thing about provisioners that I want to mention which is using provisoners are not the recommended way according to Terraform bcs Terraform doesn't have control over the Provisioner, so can not managed the State 

 - So one of the problem we have with Provisioner is a timing issue , so by the time that this gets executed, the EC2 instance may not be fully created and initialized so basically when we executed a shell command here or even some other command, we might have an issue where the command already execute before the server is even able to get the request  

In Ansible we can avoid this timing issue by controlling when to connect to the server, and that is actually very helpful if I actually have this problem and also to make sure that the server is fully accessible and it is possible to ssh into the Server by the time that Ansible playbook gets executed 

To configure that in Ansible to fix the timing issue . At the beggining before we execute anything I will add the logic or basically we are gonna add a `play` that will check that the Server is already accessible on its SSH port and then we can start executing tasks on that Server 

 - `gather_facts: False` we are not trying to gather facts at the beginning, we just to check that the SSH connection is possible

 - I will use a module `wait_for` let us configure a criterion or basically some logic that tells Ansible to wait for this logic to be True before executing the next play (https://docs.ansible.com/ansible/latest/collections/ansible/builtin/wait_for_module.html#ansible-collections-ansible-builtin-wait-for-module)

 - I can configure waiting for a Port to be open on a Server and that what's I gonna use . So basically we gonna say wait for port 22, which is ssh port to be open

 - We can add the `delay` start checking ssh port is open after 10s

 - We can also add `timeout` to 100s to basically wait for that amount of time

 - `search_regex: OpenSSH` Which basically mean, so we want the port 22 to be open but also contain open SSH

 - And important part is on which host we are waiting for port to be open `host: '{{ (ansible_ssh_host|default(ansible_host))|default(inventory_hostname) }}'` . This basically set the hosts value dynamically that we passed on . So that we don't have to hardcode any value

 - Also add `ansible_connection: Local` mean that we want execute this task locally, so from our local host we want to connect to the host that we get as a parameter and on that host we want to check these conditions

 - Also add `ansible_python_interpreter: /usr/bin/python3`

```
- name: Wait for SSH connection
  hosts: all
  gather_facts: False
  tasks:
    - name: Ensure SSH port open
      wait_for:
        port: 22
        host: '{{ (ansible_ssh_host|default(ansible_host))|default(inventory_hostname) }}'
        delay: 10
        timeout: 100
        search_regex: OpenSSH
      vars:
        ansible_connection: local
        ansible_python_interpreter: /usr/bin/python3
```

#### Using null resource

Another way to execute a provisioner if we want to separate it from AWS instance `resource` and have it as a separate task. 

In Terraform we can do that using a resource type called `null_resource`. 

`null_resource` is basically a Terraform provider that give us this `null_resource` type that we can use if we are not creating any specific resource on a cloud provider, or some other other platform, but we just want to execute some command either locally or remotely on the target machine 

```
resource "null_resource" "configure_server" {
  triggers = {
    trigger = aws_instance.myapp-server.public_ip
  }

  provisioner "local-exec" {
    working_dir= <Path-to-Ansible-Project>
    command = "ansible-playbook --inventory ${aws_instance.myapp-server.public_ip}, --private-key ${var.ssh_key_private} --user ec2-user <Ansible-deploy-file>"
  }
}
```

Now we have a separate task or resource for Ansible playbook execution, and I can use this for `remote exec`, `local exec` . But I have to adjust the `${self.public_ip}` bcs we don't have this self reference anymore, bcs we are outside of the aws instances resource . We can just easily reference it the `aws_instance resource` like this  `${aws_instance.myapp-server.public_ip}`

In `null_resource` we also have `triggers` attribute , Optional  but using `triggers` we can tell Terraform when to run `null_resource` . In our case we can configure Terraform to execute this `null_resource` or execute the Ansible Playbook acutally inside whenever a new server is created or basically whenever the ip address of that aws instance resource changes, so we know that a new server got created, so we need to run Ansible playbook

####  Fixing Ansible chmod: invalid mode A+user and Remote Temp Directory Warnings

####  Problem 1: ACL Permission Error When Using become_user

![Screenshot 2025-05-28 at 12 16 13](https://github.com/user-attachments/assets/8acc417a-8475-4fa5-a1e3-14be9a37a7bc)

Error Message : 

```
chmod: invalid mode: ‘A+user:tim:rx:allow’
Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user.
```

Cause : 

- When Ansible uses `become_user` to switch to an unprivileged user (like tim, ec2-user, etc.), it creates temporary files in `/tmp` on the remote server.

- To grant that unprivileged user permission to use these temp files, Ansible tries to apply `Access Control Lists (ACLs)` using a command like: `chmod A+user:tim:rx:allow /tmp/ansible-tmp-XYZ`

- This is not normal UNIX chmod. It's an ACL-based command that only works if:

  - The underlying filesystem supports ACLs (most do)

  - The `acl` package is installed (provides the required system commands like setfacl, getfacl, etc.)

Solution :

- Install `acl` Package on the remote Server before execute the script : `sudo yum install acl -y`

#### Problem 2: Ansible Warning About Missing ~/.ansible/tmp Directory 

![Screenshot 2025-05-28 at 12 15 59](https://github.com/user-attachments/assets/99e6a4b7-096b-4f04-84ad-e2b5046e48e0)

Warning Message : 

```
[WARNING]: Module remote_tmp /home/tim/.ansible/tmp did not exist and was created with a mode of 0700, this may cause issues when running as another user.
```

Solution: 

- Create the `.ansible/tmp` Directory in Your Playbook

```
- name: Create new Linux User
  hosts: ec2_servers
  become: yes
  vars_files: project-vars.yaml
  tasks:
    - name: Create new Linux user
      user:
        name: "{{linux_name}}"
        groups: adm, docker
        create_home: yes
    - name: Create ~/.ansible/tmp for tim
      file:
        path: /home/{{linux_name}}/.ansible/tmp
        state: directory
        owner: "{{linux_name}}"
        group: "{{linux_name}}"
        mode: '0700'
```


## Dynamic Inventory

Let say we have an infrastructure that we are managing with Anisble which is very dynamic, meaning new server get added and deleted all the time . And this is actually a very common practice when I have an infrastructure that has some auto-scaling maybe configured so whenever I have high demand for Server resources I basically scale up, I add a couple of new servers when I don't need that much resources, I scale them down 

In this case as I can imagine having a static list of inventory for Ansible doesn't make any sense, bcs IP address will change all the time .

We need some way to dynamically set the IP address or DNS names of the Servers that Ansible should manage 

Previous example we saw a case where we created a server using Terraform and then we used Terraform to hand over the control to Ansible and execute an Ansbile playbook command using that newly created Server instances, which was also an example of setting an IP address of of  the controlled node for Ansible using a Variable 

In this case we will execution of Terraform and Ansible again, so we will create 3 EC2 Instances using Terraform and then we are going to go to Ansible project to connect to those 3 Servers and configure them without hard coding the IP address of those Servers . Basically dynamically getting or fetching the information about the server IP address and using that in Ansible 

#### Terraform Create EC2 

To create 3 server I just need to replicate 3  `resources aws_instance` and give it 3 different name

For the third server I will set another `instance_type = "t2.small"`

Save it and create a Iac `terraform init` then `terraform apply`

 New server get added and removed all the time I want to fetch that information from AWS dynamically . We don't know how many Servers running on AWS account we don't know  what the IP addresses of them are, what Instance types etc . We just know that there are some servers running and we want to configure all of them using our Ansible playbooks 

 #### Inventory Plugin and Inventory Script 

For configuring a dynamic inventory for Ansible, we would need to connect to the AWS account from Ansible and fetch the information about the server instances from a specific region and get the Public IP address or public DNS name for all those Servers . 

So we need basically a functionallity in Ansible that is able to do all of that

 - Connect to AWS account and get server information

 For that we have 2 options in Ansible . (https://docs.ansible.com/ansible/latest/inventory_guide/intro_dynamic_inventory.html)

  - Dynamic inventory plugins

  - Dynamic inventory script

Ansible recommended to use Dynamic Inventory Plugins over Dynamic Iventory Script 

 - The main reason is bcs Plugins can use of Ansible Fetures like State Management

Inventory Plugis written in Yaml, Inventory Script are written in Python  

So the YAML format and the plugin functionality basically is part of the Ansible general formats and the inventory plugins also use all the Ansible Features and also the plugins use most of the recent updates of the Ansible code itself 

The script do not have that advangtages 

For both Inventory plugins and inventory scripts we have a list of them for different infrastructure providers  

 - If we connecting to an AWS account bcs we want inventory from AWS then we will need inventory plugin for AWS specifically

 - And If I am using Google or other platfor, I would need to find inventory plugin or script for that specific provider

 - In this case I will find an inventory plugin for AWS EC2  

#### AWS EC2 Plugin

(https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)

If I want to use this plugin then I have to have Python modules, installed locally on my Laptop and the I will able to use that . I also need Boto3 and Botocore install locally

Now go back to Ansible project. First thing I need to do is in the Ansible configuration `ansible.cfg`,  we need to enbale AWS EC2 plugin

 - I will use `enable_plugins` to enable aws ec2 plugins . This could be a list of plugin 

```
enable_plugins = aws_ec2
```

#### Write plugin configuration

Now in this project I will create a new file and this is going to be our plugin configuration file `inventory_aws_ec2.yaml`

This is an important part about naming of this file . The last part of the name of this file has to be `aws_ec2`. Otherwise the plugin will not be able to recognize or identiry our plugin file  

<img width="500" alt="Screenshot 2025-05-14 at 19 08 49" src="https://github.com/user-attachments/assets/e7845723-73ed-4286-b0e4-19f61658586c" />

To write the plugin configuration we need the name of the plugin which is aws ec2

Now comes the attributes or configuration for the plugin 

 - First configuration we need is which region of AWS we want to check our inventory from . We can have multiple regions 

```
---
plugin: aws_ec2
regions:
  - us-west-1
```

Example above basically we just enables the plugin and then we configure that plugin to tell it which region to connect to . With that configuration now I can execute out plugin separately without any playbook execution and see what it returns 

So we can bascially just test it to see that we get the list of EC2 Instances . And for testing just the inventory plugin there is acutally a convenient to do that using `ansible-inventory`  command . 

<img width="500" alt="Screenshot 2025-05-14 at 19 18 37" src="https://github.com/user-attachments/assets/f5323f7a-ac67-47cb-a095-90931ba2a904" />

And to do that `ansible-inventory` command will take a parameter for inventory `ansible-inventory -i <file of inventory plugin> --list`

 - `--list` use to list a result

 - Once executed I have a huge output in the terminal. We have all the data with alot of attributes for all the Servers that this inventory plugin was able to fetch from AWS .  

Important to differenciate here we are not connecting to the Server yet . We are not ssh-ing into the Servers . We are not connecting to it bcs we haven't defined the private key or the server user here . We are just connect to AWS account and basically fetching the information about the server using the Python boto module 

If I don't want to see this huge ouput of all the information about the server, we only want the ouput of the server host names . We can do `ansible-inventory -i <plugin yaml file> --graph`

 - Then I will have a cleaner output with one group which is called AWS EC2 and the 3 Server

 - This is basically the equipvelent of the entry of the IP address in the `hosts` file

 - The Result in the terminal is a DNS name which then `Playbook` would use to connect to the Instances

 #### Assign public DNS to EC2 Instances 

This is a DNS name of the Instances which are private DNS name . Not public DNS that we can use to connect to the Server 

<img width="400" alt="Screenshot 2025-05-16 at 10 51 34" src="https://github.com/user-attachments/assets/af521e2b-c90a-4bca-a30f-b90fe6de443f" />

Important thing to know about AWS EC2 Dynamic Inventory is that if our Instances do not have Public DNS name assigned, we get the private DNS name

In order to be able to connect to the Servers using the Private DNS or Private IP address, our Ansilbe command would need to be executed from inside the VPC where the Server are running

If we want to connect to the VPC from **outside** the VPC, in this case from my Laptop I would need to use a Public DNS name or Public IP address

To fix that We will change our Terraform configuration so that when it creates these 3 Servers, all of the gets assigned a public DNS name 

 - Go back to Terraform and configure my Server to get the Public DNS automatically assigned .

 - That Configuration is acutally on the VPC level and not on the Instance level

 - I will add `enable_dns_hostnames = true` to `resources "aws_vpc" ""` . Now it will take care of that problem 

Now I have the Public DNS name and this will be use in the Playbook to connect to the Server 

#### Configure Ansible to use dynamic inventory 

 Basically now I have the `hosts` file configured in `ansible.cfg` by default as a `inventory = hosts` . 

 So instead of the `hosts` file we want our `playbook` to basically use that `inventory plugin` whenever playbook gets executed 

To do that . First in the `Playbook` we need to have a reference to the `host` that the plugin actually returns 

 - We can do either `all` that we used previously when we exected Ansible from Terraform project

 - Or we acutally have a group name that we get by default, which is `aws_ec2`
   
<img width="400" alt="Screenshot 2025-05-16 at 11 21 48" src="https://github.com/user-attachments/assets/fdb6f978-4624-4a54-9667-252faecd7c6a" />

 - We can use `aws_ec2` group name in the `hosts` instead of `all` and this will set the hosts to all the value that AWS EC2 group contains  

And second thing is whenever we are connecting to the `hosts` we need a username and the Private SSH key 

And if we want them set globally we can configure them inside the Ansible configuration file `ansible.cfg` . So we could pass those the user and private key file as parameters to Ansible Playbook command like we did in the Terraform configuration . Or if we want to spare us setting them as parameters we can just globally configure them 

So in `ansible.cfg` I will set `remote_user = ec2-user` and `private_key_fike = ~/.ssh/id_rsa`

So we have everything ready and configured to execute `Playbook` on those 3 servers . `ansible-playbook deploy-docker-new-user.yaml` . 

And we need to tell Ansible do not take `hosts` file but take the `inventory_aws_ec2.yaml` . The complete command will look like this : `ansible-playbook -i inventory_aws_ec2.yaml deploy-docker-new-user.yaml`

If we decide we alway gonna use the Inventory plugin as a Inventory source for our `Playbook` we could make `inventory_aws_ec2.yaml` as a default `hosts` file  like this in `ansible.cfg` -> `inventory = inventory_aws_ec2.yaml`

#### Target only Specific Servers

Now let say we don't want to execute the `Playbook` on all the Server we may want just target on some of the Server and not all of them 

 In terraform I will configure 2 dev server and 2 prod server as a Tag 

Imagine we may have 10 development Servers, 20 Staging Servers and 20 Production Servers . We may want to execute different `Playbooks` on different types of Servers . 

We may have a `playbook` for development server and slightly different one on production server 

To configure this dynamic plugin to give us only the `dev servers` or only the `prod servers` and not just everything from that region 

 - In the plugin configuration I can use `filters` to filter the Server I want to get as a result instead of just getting all of them by default .

 - And to know the attributes that I can use as filters I can actually reference the list (http://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html#options.) right here

 - I can use any of that attribute as a filter . In our case we have tag name configure for each server . So we want to filter base on the tag name .

```
filters:
  tag:Name: dev*
```

 - The configuration above will give me a server with the tag name `dev`

 - To test that I can execute `ansible-inventory -i inventory_aws_ec2.yaml --graph`

 #### Create Dynamic Groups 

Now it could be that using that inventory plugin, we want to acutally return both Dev server and production server, but we want to differentiate them . We want to group them into separate group 

Intead of having everything in AWS EC2 groups, we want to have own group for `dev servers`, own group for `production servers`

And for that we can use another configuration which is called `keyed_group`

 - `keyed_group` actually has a list of configuration
 
   - `key` : What do we want or what attribute of the information of the instances do we want to use for grouping . In this case I want to use `tags`. `key: tags`
  
   - The different in the `keyed_groups` we use the key names, which are one of the attributes that we have . For the filters we used a different attribute lit provided by filters configuration
  
   - For `keyef_groups`. We actually look for the attributes that we get on the Instances
  
   - With this configuration we gonna get additional group that are group by `tags`. So we have 1 tag for each server which is name tag . So we gonna get 2 groups, one for developed Server and one for production Server
  
```
plugin: aws_ec2
regions:
  - us-west-1
keyed_groups:
  - key: tags
```

After executed that we still have common list where all the servers are listed so we can address all the servers with this group . And in addtion to that we have separate one for `_Name_dev_server` and `_Name_prod_server`

If we don't want to start with underscore we can add `prefix`

```
plugin: aws_ec2
regions:
  - us-west-1
keyed_groups:
  - key: tags
    prefix: "tag"
```

That mean if I want to execute this `playbook` only for the dev servers, I will going to copy the name of the group and set that as `hosts` value instead of the whole AWS EC2 lists

Now I can execute the playbook `ansible-playbook deploy-docker-new-user.yaml`

Now we know how to target server by grouping them together, and `tags` is just one of the keys that I can use to group instances . I can also use the image name and other attributes and I can acutally create multiple group . So we can leave this with the text, the grouping with text and we can add another one .

Let say in addition I want to froup the server based on their instance type 

 - I will take the attribute call `instance_type`

```
plugin: aws_ec2
regions:
  - us-west-1
keyed_groups:
  - key: tags
    prefix: "tag"
  - key: instance_type
    prefix: instance_type
```
