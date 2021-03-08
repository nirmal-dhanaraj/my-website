---
title: "Orchestrating CyberArk PAM Deployments Using Ansible"
comments: true
last_modified_at: 2021-03-09T08:20:02-05:00
show_date: true
related: true
share: true
categories:
  - Blog
tags:
  - CyberArk
  - PAS-Orchestrator
  - PAM
  - CyberArk Automation
---

 <br>
![Eclipse](/assets/images/blogs/orchest/design1.png)
<br><br>
Its been a while I’m thinking about writing a blog about the deployment options in CyberArk PAM. If it is a Cloud only environment there is a rich set of CloudFormation templates that enables easy deployment on AWS. Azure Resource Manager (ARM) templates enable easy deployment on Azure. However, it is difficult to combine the different methods to orchestrate and automate a hybrid deployment

CyberArk PAS Orchestrator is a set of Ansible roles which provides a one stop solution to deploying CyberArk Core PAS components simultaneously in multiple environments, regardless of the environment’s location.

CyberArk provides a set of Ansible roles to orchestrate and
automate CyberArk components deployment. These Ansible Playbooks are publicly available in the
CyberArk GitHub repository.

In simple terms, the whole PAS Orchestrator can be deployed in a single server (Unix/Linux or Windows) and this will take care your entire PAS compoenents on the target windows machines. 

## Requirements
- Unix VM  ==> I deployed a simple ubuntu linux for this lab. Playboook will be executed in this box.      	  CyberArk components CD image must be copied into this server.
- Ansible 
- Python 3.4
- PywinRM ( I have configured kerbrose authentication for this lab)
- The remote host must have Network connectivity to the CyberArk vault and the repository server
	---> 443 port outbound
	---> 443 port outbound (for PVWA only)
	---> 1858 port outbound
- Administrator access to the remote host (PVWA, CPM and PSM)

Each PAS component’s Ansible role is responsible for the component's end to end deployment, which includes the following stages for each component:

1. Copy the installation files to the target windows server
2. Install the prerequisites
3. Silent installation of the component
4. Post installation procedure and hardening
5. Registration in the Vault

## Installation Steps

Step 1 : Logon to the Linux server and install Ansible by executing the below command 
```ruby
sudo apt-add-repository ppa:ansible/ansible
sudo apt update 
```
![Eclipse](/assets/images/blogs/orchest/2ansible.png)
<br>

Step 2 : Import the Ansible Role from Github
```ruby
git clone https://github.com/cyberark/pas-orchestrator.git

```
 <br>
![Eclipse](/assets/images/blogs/orchest/1.import.png)
<br>
Step 3 : Execute the below command in order to get the components roles
```ruby
ansible-galaxy install --roles-path ./roles --role-file requirements.yml
```
```
Step 4 : This step is required to Setup kerbrose authentication betweeen Linux and target windows machine
sudo apt-get install python-dev libkrb5-dev krb5-user
sudo pip install kerberos
```
Step 4.1 : Edit /etc/krb5.conf file to include target hostnames as follows 
```ruby
[libdefaults]
default_realm =  CORPAD.COM
[realms]
 CORPAD.COM.COM = {
 kdc = pam-comp-srv.corpad.com
 kdc = pam-cpm-srv.corpad.com
 admin_server = pam-comp-srv.corpad.com
 admin_server = pam-cpm-srv.corpad.com
 }

[domain_realm]
 .corpad.com = CORPAD.COM
 corpad.com =  CORPAD.COM
```

Step 5 : Edit the /etc/ansible/hosts file and include the below target details 
```ruby
[PAS]
pam-comp-srv.corpad.com
pam-cpm-srv.corpad.com

[PAS:vars]
ansible_user=ndhanaraj@CORPAD.COM
ansible_password=Password
ansible_winrm_transport=kerberos
ansible_port=5986
ansible_connection=winrm
ansible_winrm_server_cert_validation=ignore
```
You can validate the above authentication steps using the below command 
```ruby
ansible pam-comp-srv.corpad.com -m win_ping -vvvvv
```
 <br>
![Eclipse](/assets/images/blogs/orchest/5.1check.png)
<br>
Step 6 : Copy the CyberArk CDs (pvwa, cpm & PSM into linux machine under /tmp/ directory)

Step 7 : Execute the below powershell script on the target component servers. This will enable winRM service in windows servers. This is the only thing we need to perform on all the remote servers .
```ruby
ConfigureRemotingForAnsible.ps1

Note : if you want setup to ansible server without Kernbrose auth
	ConfigureRemotingForAnsible.ps1 –EnableCredSSP
```
 <br>
![Eclipse](/assets/images/blogs/orchest/remote.png)
<br>

When every thing goes well ,you can run spring boot applications and if there is no error it will automatically retrieve the status of all the instances of each component and If it finds 
any component id is down, then this custom solution will create a ServiceNow Incident record. 
<br>
Step 8 : Finally we need to run the Ansible palybook. In my example I'm deploying a PVWA and CPM services using the below commands 
```ruby

ansible-playbook -i ./inventories/staging pas-orchestrator.yml -e "vault_ip=10.1.3.10 ansible_user=ndhanaraj@CORPAD.COM pvwa_zip_file_path=/tmp/pvwa.zip cpm_zip_file_path=/tmp/cpm.zip connect_with_rdp=Yes accept_eula=yes cpm_username=WinManager"
```

 <br>
![Eclipse](/assets/images/blogs/orchest/run.png)
<br>
If you encounter any issues you will get the error message in the screen as well as you can find the relevent logs under the dedicated log directory.  In my case everything went well and my console looks like below 
 <br>
![Eclipse](/assets/images/blogs/orchest/outcome.png)
<br><br>
Step 9 : Now its time for validating the component services status. Now Im able to login to PVWA web portal and im able to see both web app and CPM services are up and running.
 <br>
![Eclipse](/assets/images/blogs/orchest/webapp.png)
<br><br>
I've verified my CPM service name since I used custom name while I run my playbook and same will be reflecting in the below screenshot.
 <br>
![Eclipse](/assets/images/blogs/orchest/ark.png)
<br>
## Summary 
I’ve deployed this in a latest ubuntu server on Azure and the instllation steps for other types may vary a bit but the concept remains same. Im not a devops guy but it take 30 minutes to set up this and then ran the playbook. This is very handy for medium and large PAM deployments since it supports paralel installation on multiple remote servers. The Ansible roles can be integrated with the organization’s CI/CD pipeline.