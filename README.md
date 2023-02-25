
# Chekout EC2 Dashboard here 

<img width="692" alt="Ec2 dashboard" src="https://user-images.githubusercontent.com/93249038/221330932-b963b05a-b9f2-4a4e-84ab-037a11d82db2.png">

This project contains the Ansible inventories, playbooks, and templates needed to manage EC2 instances and individual applications.
# Create the IAM User
Under the IAM section go to Users and the click on Add user to create a user specifically for this role. In my case, I named it admin and set Access type as shown below:

Add User

Next, attach a policy broad enough to allow instance creation/modification:

Attach Policy

Finally, create and download the generated access keys (these will be saved to a credentials.csv file) to be used with the playbook:

# Download Keys

Install Ansible
The Ansible Playbooks will be run from a control server (in my case, a MacBook Pro) that is ssh enabled. They are of course checked into this git repository under the ansible directory.

Ansible Installation
Review your OS-specific installation instructions. For the Mac OS I used the below commands:

sudo pip install --upgrade pip
sudo pip install ansible
The second command installs the following packages: MarkupSafe-1.1.1 ansible-2.9.7 cffi-1.14.0 cryptography-2.9.2 enum34-1.1.10 ipaddress-1.0.23 jinja2-2.11.2 pycparser-2.20

Note that I am using Python 2.7.x which will work for now but is officially not longer supported as of 2020.

You can also run ansible –version to get specific information about the installation:

ansible 2.9.7
  config file = None
  configured module search path = [u'/Users/stuartpineo/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /Library/Python/2.7/site-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 2.7.10 (default, Jul 15 2017, 17:16:57) [GCC 4.2.1 Compatible Apple LLVM 9.0.0 (clang-900.0.31)]v
In addition to Ansible we will use boto and boto3, Python interfaces to AWS implementing the ec2 and ec2_group modules (installation/dependencies shown below):

pip install boto boto3
...
Successfully installed boto-2.49.0 boto3-1.13.8 botocore-1.16.8 s3transfer-0.3.3 urllib3-1.25.
Then create the .boto file in your home directory with the below contents (this is the access key information you downloaded to the CSV file) ensuring that the file permissions are set to 400 (alternatively, you can also reference these keys in your playbook as I will show):

[Credentials]
aws_access_key_id = YOURACCESSKEY
aws_secret_access_key = YOURSECRETKEY
Spin up an AWS Instance: Configure the Inventory File and Run the EC2 Instance Playbook
The ansible directory in this repository contains the playbooks/ec2_instance.yml generic playbook to spin up the instance and the inventories/ec2_instance.template that can be used to create the inventory file.

Once the inventory file (i.e., inventories/ec2_instance) values are in place from the ansible subdirectory run the command (output follows):

ansible-playbook -i inventories/ec2_instance playbooks/ec2_instance.yml

PLAY [EC2 Instance] ****************************************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************
ok: [localhost]

TASK [Launch EC2 Instance] *********************************************************************************************************************************************************************
changed: [localhost]

PLAY RECAP *************************************************************************************************************************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
Navigating to the EC2 Dashboard you can check the status of the instance (in this case TestInstance):

Running Instance

Since I spun up an Ubuntu instance, I can now access this instance from an SSH-enabled terminal using the ubuntu user:

ssh -i /keypath/mykey.pem ubuntu@ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com
Ansible Configuration: Automated AccumuloConfiguration/Start-up of Zookeeper, Hadoop, and Accumulo
Create the EC2 Security Group
To create/update the ‘ZK-HA-AC-Restrictive’ security group (assuming the instances are already running) first create a inventory file from the template and then run the playbook. From the ansible subdirectory:

ansible-playbook -i ./inventories/zk_ha_ac_conf ./playbooks/zk_ha_ac_group.yml
Start/Stop the Instances
The instance_id for the cluster instances should be added to the zk_ha_ac_config inventory file. The playbooks start and stop can then be run to start/stop the cluster instances:

ansible-playbook -i ./inventories/zk_ha_ac_conf ./playbooks/zk_ha_ac_start.yml
ansible-playbook -i ./inventories/zk_ha_ac_conf ./playbooks/zk_ha_ac_stop.yml
Stop/Starting Instances and attaching the Security Group
The ‘instances’ playbook initializes the security group, stops/starts the instances (attaching the security group), and adds the restrictive rules to the security group based on the new instance information (such as new public IPs). To set this up, modify the zk_ha_ac_conf inventory template and then run the playbook:

ansible-playbook -i ./inventories/zk_ha_ac_conf ./playbooks/zk_ha_ac_instances.yml
Generating the ‘servers’ File
The servers playbook queries the running EC2 instances (based on instance ids) for the public/private DNS names and generates the ‘servers’ file in the inventories directory. Execute using the same inventory file used previously:

ansible-playbook -i ./inventories/zk_ha_ac_conf ./playbooks/zk_ha_ac_servers.yml
Generate the ‘.aliases’ File (optional)
This file gets sourced by the .bashrc to create simple to invoke aliases to SSH into the servers and create SSH tunnels to access the Web applications (since my own cluster is, under optimal conditions, locked down to use exclusively private DNS names leaving only port 22 opened externally to the /32 network mask)

Running the Application Playbooks
Before starting, you will need to create an ansible_hosts (or whatever name you choose) inventory file which you can copy/modify from the ansible/inventories/ansible_hosts.template checked into this repository (the server values are the Public DNS or IP). In addition, the server file should include the hosts (it can be generated dynamically). You probably will not need to modify any of the playbooks or templates though its probably good to review them before a deployment in case you need to extend or modify the configuration.

Once done, you can test it by running ansible servers -m ping -i ./inventories/servers to verify that instances are up. If successful, you should output similar to the one below for each instance pinged:

ec2-xxx-xxx-xxx-xx1.compute-1.amazonaws.com | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
To run the zookeeper_conf playbook which ensures that the configuration contains the right hosts (and other values listed) as well as restarting the daemons (zookeeper_daemons on each host run below commands:

cd ansible
ansible-playbook  -i ./inventories/servers -i ./inventories/ansible_hosts ./playbooks/zookeeper_conf.yml
ansible-playbook  -i ./inventories/servers -i ./inventories/ansible_hosts ./playbooks/zookeeper_daemons.yml
You will see output associated with each task, but if all goes well will get a play recap at the end showing the successful completion status.

PLAY RECAP *************************************************************************************************************************************************************************************
ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com : ok=6    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
Similarly to run the hadoop playbook (which is primarily template driven) execute:

ansible-playbook -i ./inventories/servers -i ./inventories/ansible_hosts ./playbooks/hadoop_conf.yml
To start up DFS and YARN daemons there is a separate playbook that can be executed:

ansible-playbook -i ./inventories/servers -i ./inventories/ansible_hosts ./playbooks/hadoop_daemons.yml
The hadoop playbook not only overwrites the configuration files listed in the playbook but also sets up the SSH configuration and executes the commands to stop/start DFS and YARN.

To run the accumulo playbook execute:

ansible-playbook  -i ./inventories/servers -i ./inventories/ansible_hosts ./playbooks/accumulo_conf.yml
To run the Accumulo cluster daemons stop/start playbook execute:

ansible-playbook  -i ./inventories/servers -i ./inventories/ansible_hosts ./playbooks/accumulo_daemons.yml
Playbook Run Automation
In the $ANSIBLE_HOME/bin directory the script run_playbooks.pl will initiate the sequence of playbook actions needed to start the cluster instances (which includes configuring the applications and restarting their daemons) or stop the cluster instances.

Typing run_playbook.pl –help will shown usage and examples:

Message : help
Usage   : ./run_playbooks.pl --ansible-home <absolute or relative path> [ --start --stop  --debug --verbose ]
Examples: ./run_playbooks.pl --ansible-home .. --debug   # Start up the instances and configure/run the cluster (prompt driven)
          ./run_playbooks.pl --ansible-home .. --start   # Same as previous command without the --debug
          ./run_playbooks.pl -=ansible-home .. --stop    # Stop the cluster (prompt driven)
The script execution allows you to skip a playbook run (by hitting ‘n’ or return key), re-try (if run failed initially), or exit the playbooks runs all together.

Ansible References
https://docs.ansible.com/ansible/latest/modules/ec2_module.html
https://docs.ansible.com/ansible/latest/modules/ec2_instance_module.html
https://docs.ansible.com/ansible/latest/modules/ec2_group_module.html
https://docs.ansible.com/ansible/latest/modules/ec2_instance_info_module.html
https://docs.ansible.com/ansible/2.4/playbooks_loops.html#nested-loops
https://docs.ansible.com/ansible/latest/modules/lineinfile_module.html
https://docs.ansible.com/ansible/latest/modules/debug_module.html
