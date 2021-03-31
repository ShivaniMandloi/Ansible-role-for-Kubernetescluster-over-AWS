# Ansible-role-for-Kubernetescluster-over-AWS
<center>Though containers are a good way to bundle and run your applications.<br>
In a production environment, you need to manage the containers that run the applications and ensure that there is no downtime.<br> 
For example, if a container goes down, another container needs to start but doing this manually is would be a very difficult task.<br>
</center>
<h3>Wouldn’t it be easier if this behavior was handled by a system?</h3>

<b><i>  
That’s how Kubernetes comes to the rescue! 
Kubernetes provides you with a framework to run distributed systems resiliently. 
It takes care of scaling and failover for your application, provides deployment patterns, and more.
For example, Kubernetes can easily manage a canary deployment for your system.
</i></b>

What is the Kubernetes cluster?
A Kubernetes coordinates a highly available cluster of computers that are connected to work as a single unit that is a Kubernetes cluster.
it is actually a set of nodes that run containerized applications. Containerizing applications package an app with its dependencies and some necessary services. Kubernetes clusters allow for applications to be more easily developed, moved, and managed.
This cluster can be created on any operating system either is on a physical machine, virtual machine, container, or cloud
It is comprised of one master node and several worker nodes.
master node controls the state of the cluster.
it takes care of the following responsibilities like applications are running and their corresponding container images. assigning the task to the suitable node with respectively, It coordinates processes such as:
Scheduling and scaling applications
Maintaining a cluster’s state
Implementing updates

Here, our task is to create this Kubernetes cluster over AWS instances using an ansible role. there are prerequisites before we create a cluster on AWS
we can launch ec2 instance in different ways like
Console password: A password that the user can type to sign in to interactive sessions such as the AWS Management Console.
Access keys: A combination of an access key ID and a secret access key. You can assign two to a user at a time. These can be used to make programmatic calls to AWS. For example, you might use access keys when using the API for code or at a command prompt when using the AWS CLI or the AWS PowerShell tools.
since we are using ansible we will need Access key ID and secret access key
You can create these by creating an IAM user with a power user role and selecting the programmatic access option and download your credentials.
also, you will need a pre-created key pair while launching an instance.
you can get it in AWS ec2 console, create a key pair and download it and import it to your Linux computer using WinScp.
now let’s start with the task
Creating a workspace with name kubernetesclustersetup
Creating Ansible role ec2setup for launching ec2 instance
create a role using command
#ansible-galaxy init <role_name>
Now, we will write our tasks in tasks/main.yml
for this role we need
AMI image id
Security group id
keypair name you downloaded earlier
Accesskey Id and Secret key
so, to tell our role what task it should do you have to write it in ec2setup/tasks/main.yml this is the file where role checks all its tasks

the task to launch 1 master node in AWS
Here, I have set count 1 since we have to create a master node.

also, I have created variables for count(No. of worker node you want ), AMI image, security group id, key, access key, and secret key so we write variable in Jinja format {{ variable }}.
so while execution ansible goes search for its value in vars folder of roles and take that value.
and you can define these variables in ec2setup/vars/main.yml as follows

img: var file for ec2setup
since Access key Id and secret key are highly sensitive data we should secure than for this we have ansible-vaults
•What is Ansible-vault?
Ansible Vault encrypts variables and files so you can protect sensitive content such as passwords or keys rather than leaving it visible as plaintext in playbooks or roles. To use Ansible Vault you need one or more passwords to encrypt and decrypt the content. If you store your vault passwords in a third-party tool such as a secret manager, you need a script to access them. Use the passwords with the ansible-vault command-line tool to create and view encrypted variables, create encrypted files, encrypt existing files, or edit, re-key, or decrypt files.
Creating vault and storing the AWS access key and secret key there using command

As you enter it asks for setting a password make sure you create this vault in the same workspace.
Now creating Playbook to call this role for localhost

since you have created a vault for the access key and secret key you need to mention it in your main playbook using keyword
vars_files:
but how playbook will get to know that you have secured it so while running playbook you should mention your vault id then first it asks us to enter the vault password and then executes.
Since our playbook will go on our behalf to our AWS account we need to give privilege escalation.
What is privilege escalation?
Ansible uses existing privilege escalation systems to execute tasks with root privileges or with another user’s permissions. Because this feature allows you to ‘become’ another user, different from the user that logged into the machine (remote user), we call it become.
so, go in ansible configuration file
/etc/ansible/ansible.cfg

Note: since we want to configure Kubernetes on the launched instances we need their public IP. but, every time you launch or start an instance its IP Address changes so we cannot put them in a static way, we need some program that would itself go to our AWS account and retrieve IP from there, this program is nothing but dynamic inventory.
downloading this pre-created dynamic inventory using python.
this inventory needs a boto3 module. using which it automatically goes to our AWS account and retrieves the IP-Address . for which we need to give our access key and secret key, we can simply save them in an environment variable.


copy these files in a single folder like /etc/mydinv and give this path in the ansible configuration file you can refer above configuration image.


Now let’s run our playbook for ec2setup

You can check your dynamic inventory working or not using command
#./ec2.py
this command shows the various groups created for example this


2. Now, let’s create the role for the Kubernetes Master node and worker node
using command
#ansible-galaxy init Masternode
#ansible-galaxy init Workernode
let’s assign tasks in Masternode/tasks/main.yml
Steps:
Since Kubernetes launch pods over docker, we need to configure our node with docker to install, start and enable docker service.
Now, to configure the master node for the Kubernetes cluster first we need to configure its yum repository
since we are creating a cluster using the kubeadm program
Kubeadm is a tool built to provide kubeadm init and kubeadm join as “fast paths” for creating Kubernetes clusters. so we need to install it.
with this, also installs kubelet and kubectl program
Kubectl command-line tool lets you control Kubernetes clusters.
Kubelet is the primary “node agent” that runs on each node and is used to register the nodes in the Kubernetes cluster through the API server. After a successful registration, the primary role of kubelet is to create pods and listen to the API server for instructions.
4. enable kubeadm
5. pulling kubeadm config image
6. Changing the settings such that your container runtime and kubelet use systemd as the cgroup driver stabilized the system. To configure this for Docker, set native.cgroupdriver=systemd.uses systems driver wherein docker creating daemon.json file in /etc/docker
for which I have created a file in template first as follows

7. Restarting docker
8. Setting permissions
9. Starting kubeadm service
10.Copying config file for k8s
11. Creating a flannel overlay network
12. Generating token to connect with worker node


Note: I have added generating token and join the token task in my main playbook but prefer to add it in your roles.
Similarly in Workernode/tasks/main.yml
note: copy the daemon.json file in the worker node template folder


Now creating a final playbook to call these roles

Now, here we have one issue that is
we save our token in a variable but we cannot directly pass the variable of one host playbook to the other host playbook so,
you can do two things either copying the token and prompt this token while joining the worker node but this is a manual process to completely automate it
we can use the concept of adding host in in-memory inventory using the module “add_host” where you can
Use variables to create new hosts and groups in inventory for use in later plays of the same playbook
Takes variables so you can define the new hosts more fully like I have used “link”.
here I have created a dummy host as linkforjoining and set a variable “link” with a variable in which we registered token. i.e “token[‘stdout’]”
Now, you can call this in worker node task since we save it in the host variable
Now, let’s run the final playbook


You can see the token is created


Successfully created Kubernetes cluster you can check by doing ssh to your master node and using command
#kubectl get node

DONE…!!!!
Hope you all find it interesting…!!!
Thankyou…✌
Shivani Mandloi
10



10 
