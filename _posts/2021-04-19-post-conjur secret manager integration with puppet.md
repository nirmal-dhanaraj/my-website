---
title: "Conjur Secret Manager Integration with Puppet"
comments: true
last_modified_at: 2021-04-19T07:07:02-05:00
show_date: true
related: true
share: true
categories:
  - Blog
tags:
  - CyberArk
  - Conjur Secret Manager
  - PAM
  - Conjur and Puppet
---

 <br>
![Eclipse](/assets/images/blogs/puppet/wf.png)
<br><br>
I have been dealing with credential provider for a long time but now one of my customers is looking for options to secure their DevOps platform, Puppet. This is one of the leading modern Configuration Management tool that is used for deploying, configuring, and managing servers and supports Dynamic scaling-up and scaling-down of machines.

In this post I am focusing on how we can integrate Conjur Secret Manager with puppet. There is a Puppet module available from Conjur, a robust identity and access management platform. This module simplifies the operations involved in establishing a Conjur host identity and allows authorized Puppet nodes to fetch secrets from Conjur.

This integration ensures security tools run separately from developer tools – to isolate secrets management from application code


## Prerequisites
For this post I’ve setup a Conjur (DAP 12) standalone to keep simple and since DAP comes as a docker container I have done the following in my Azure environment  :<br>
- Provisioned an Ubuntu Machine<br>
- Loaded DAP docker images downloaded from CyberArk Portal ( For this you know need to talk to your CyberArk representative and get the required access or you can use Conjur Open Source but there are some limitations, but it is up to you)<br>
- Load Conjur CLI : I used docker image for the CLI, but we can also ruby gem as well.<br>
- Install and configure DAP and import certificate. I used self-signed certificate with the help of open SSL.<br>

Puppet Enterprise:
  - I have installed Puppet master and two nodes (a Linux and a windows node for this test). There are plenty of step-by-step guides available on the internet for this setup.
  - The generated Self Signed Certificate must be copied to Puppet master and all the node for enabling SSL connection to DAP Server..

## Integration Approaches

1) Manifest-Provided Identit <br>
2) Pre-Provisioned Identity

First, I've installed Conjur/Dap Puppet module on the Puppet master server. I have installed the latest version 3.1.0 by executing the below command
```ruby
      puppet module install cyberark-conjur
```
The below screenshot represents Puppet admin UI where I've highlighted the master and two node servers (windows and Linux)
 <br>
![Eclipse](/assets/images/blogs/puppet/pupui.png)
<br><br>      
The below screenshot represents outcome of docker ps command on DAP server and it shows two running containers in docker engine (DAP and CLI)
 <br>
![Eclipse](/assets/images/blogs/puppet/dapstatus.png)
<br><br>  


<u><b>Manifest-Provided Identity (Linux Nodes) : </b> </u><br>
Step 1 : Create Required Policy, host, variables 
For this demo I have created simple policy and uploaded into DAP  through CLI. These policies are written in simple yml file and provides a way to organize the resources such as hosts, users, and secrets in a logical way.  This creates a secret and assign required permissions to host (node-01). This simple policy load command’s output as shown below:
```ruby
Loaded policy 'dev/srv/nix'
{
  "created_roles": {
    "dev:host:dev/srv/nix/node-01": {
      "id": "dev:host:dev/srv/nix/node-01",
      "api_key": "3j0f7w4f8g2973akxeegpk5v4j3q0k5vd2q0q85r15hnrz83qd2a3n"
    }
  },
  "version": 1
}
```
We can validate the above steps either via CLI or through DAP UI as follows. 
 <br>
![Eclipse](/assets/images/blogs/puppet/nixpolicy.png)
<br><br>  

Step 2 :
Now I’m defining a manifest file (site.pp) which is located under /etc/puppetlabs/code/environments/production/manifests/ directory. The Pupppet's conjur module provides a conjur::secret Deferred function that can be used to retrieve secrets from DAP. This manifest is very simple, and we need to make sure self-signed certificate presents in the client nodes. In this example I am simply writing the credentials in a file called secKey.txt.

```ruby
node default {
}
node /^node-.*$/ {
$dbpass = Deferred(conjur::secret, ['dev/db-password', {
  appliance_url => "https://dap-standalone.corpad.com",
  account => "dev",
  authn_login => "host/dev/srv/nix/node-01",
  authn_api_key => Sensitive("3j0f7w4f8g2973akxeegpk5v4j3q0k5vd2q0q85r15hnrz83qd2a3n"),
  ssl_certificate => file("/certs/conjur.pem"),
}])

$dbpassword = Deferred(conjur::secret, ['dev/db-password'])
  file { '/tmp/logs/secKey.txt':
    ensure => file,
    content => Sensitive($dbpassword),
    mode => '0600'
  }

}
```
Step 3 : The Puppet agent starts (manually or automatic) an agent run and communicates with the Puppet server to get the catalog data. Now im  executing the agents node manually using the below commands :
```ruby
  Puppet agent -t
```
<br>
![Eclipse](/assets/images/blogs/puppet/agentr1.png)
<br><br> 

<u><b>Pre-Provisioned Identity (Linux Nodes) : </b> </u><br>

Step 1: 
In this approach one line command in enough to retrieve the secrets from DAP and the below command should be defined in site.pp file
```ruby
$dbpassword = Deferred(conjur::secret, ['dev/db-password'])
```
In this demo for simplicity Im writing the password in to a file and puppet manifest will defined as follows :
 ```ruby

node default {
}

# Sample node with hardcode secret
node /^node-.*$/ {

$dbpassword = Deferred(conjur::secret, ['dev/db-password'])
  file { '/tmp/logs/secKeyFile.txt':
    ensure => file,
    content => Sensitive($dbpassword),
    mode => '0600'
  }
}
```
Step 2 : Now the agent node (node-01) will be configured with Conjur host identity, I will add the Conjur host and API key to Conjur identity files /etc/conjur.conf and /etc/conjur.identity.
<br><u>conjur.conf</u>
```ruby
account: dev  // Conjur Account Name
plugins: []
appliance_url: https://dap-standalone.corpad.com  // Conjur URL
cert_file: "/certs/conjur.pem" // Certificate file : Read from the Puppet agent
```
<br><u>conjur.identity</u>
```ruby
machine dap-standalone.corpad.com
login host/dev/srv/nix/node-01
password 3j0f7w4f8g2973akxeegpk5v4j3q0k5vd2q0q85r15hnrz83qd2a3n
```
Step 3: The Puppet agent starts (manually or automatic) an agent run and communicates with the Puppet server to get the catalog data. Now im executing the agent's node manually
 <br>
![Eclipse](/assets/images/blogs/puppet/agentr2.png)
<br>
## Summary 
Puppet and CyberArk Conjur integration enables organizations to provide better security and increase developer and operations autonomy using infrastructure-as-code and
security-policy-as-code. 
<br><u>Benifits:</u><br>
• Automate Puppet Dev-ops Credentials Workflow<br>
• Abstract secrets from code<br>
• Module supports Encryption and fulll secret orations<br>
• Enforce Least Privileges<br>
• Complete Audit Trail<br>
