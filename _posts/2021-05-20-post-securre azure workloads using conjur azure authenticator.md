---
title: "Secure Azure Workloads using Conjur Azure Authenticator"
comments: true
last_modified_at: 2021-05-20T07:07:02-05:00
show_date: true
related: true
share: true
categories:
  - Blog
tags:
  - Conjur
  - CyberArk
  - PAM
  - Azure Authenticator
---

 <br>
![Eclipse](/assets/images/blogs/azauth/1design.png)
<br><br>
In this post I'm going to walk you through the Conjur Azure authenticator and how it helps Azure workloads to securely retrieve  the credentials from Conjur without using any hardcoded credentials.
> I assume that you already have some basic knowledge around CyberArk Conjur,
 Machine identity, Azure IMDS service and REST API calls.

## Overview 
The Conjur Azure Authenticator is a highly secure method for authenticating Azure workloads to Conjur using their underlying Microsoft Azure Instance Metadata Service (IMDS) which is used to retrieve the metadata of the Azure virtual machine and the Azure services details by running API calls.

There are two approaches available here to enable this secure authentication :<br>
 - In the first approach you can map a set of azure workloads with one conjur identity by mentioning a subscription ID and resource group properties.
  - In the second approach you can associate each individual Azure workload with  a unique Conjur identity by enabling its user-assigned or system-assigned identities.


## Supported Azure Services 
 •  Azure Virtual Machines<br>
 •  Azure App Services<br>
 •  Azure Functions<br>
 •  Azure Container Instances
## Prerequisite
- Define Policies, Host, Secrets, Groups and permissions as per your requirements. This can be done via Conjur CLI or REST API calls.
- Enable Azure Authenticator and restart the Conjur Service 
- Enable Azure System assigned Identity or User assigned Identity via Azure Portal. In this example I am initiating secret request from my Azure VM(AD), so I've enabled System assigned Identity via Portal and make a note of the Object ID
- Get ready with the following Azure Details : Provider URI, Subscription ID, Resource group name. Below are the artifacts I used for this demonstration. You can modify them as per your requirements.

## Implementation - Part1
Step 1 : Enable Azure Authenticator by including the below (authn-azure) on the file conjur.conf which is located as mentioned below on the dcoker container. Here AzureAE1 dev is the service ID
```ruby
CONJUR_AUTHENTICATORS=authn-azure/AzureAE1
# docker exec conjur-master cat /opt/conjur/etc/conjur.conf
# sv restart conjur
```
 Note : we need to restart Conjur Service to eanble Azure Authenticator feature.

Step 2 : Define the Azure Authenticator in policy, and detail a group of Conjur hosts (applications) that have permission to use the Azure Authenticator to authenticate with Conjur.
```ruby
# authn-azure-AzureAE1.yml  
# CLI Command to load the policy : conjur policy load root authn-azure-AzureAE1.yml
----------------------------------
- !policy
  id: conjur/authn-azure/AzureAE1
  body:
  - !webservice
   - !variable
    id: provider-uri 
       
  - !group 
    id: apps
    annotations:
      description: Group of hosts that can authenticate using the authn-azure/AzureWS1 authenticator
   
  - !permit
    role: !group apps
    privilege: [ read, authenticate ]
    resource: !webservice
```
Step 3 : Populate the provider_uri variable with the provider_uri from Azure

```ruby
conjur variable values add conjur/authn-azure/AzureAE1/provider-uri https://sts.windows.net/23ae4333-8a61-444a-a59a-999999efc

```

Step 4 : In this step, you give the Azure instance an identity in Conjur, and define the application identity for each Azure instance in Conjur in the host annotations.<br>
In this demo Im going to provid the actual hostname of the VM : *AD* <br>
Also I'm going to use all the information I collected from Azure for ex: subscription-id, resource-group and group ID of the user-assigned-identity

```ruby
authn-azure-AzureAE1-hosts.yml,
------------------------------
- !policy
  id: azure-apps
  body:
    - !group
 
    - &hosts
      - !host
        id: AD # azureVM name 
        annotations:
          authn-azure/subscription-id: 034344345d-6t1b-461f-8aab-533434433255
		  # I'm using fake subscription id here 
          authn-azure/resource-group: nirmal-dhanaraj-rg
		  # This is my Resource group where the VM(AD) is located 
          # authn-azure/user-assigned-identity: <user-assigned managed identity>
          authn-azure/system-assigned-identity: 0000aaaa-00aa-00aa-00aa-00000aaaaa
		  # the above details we can get it from Azure Portal 
          
    - !grant
      role: !group
      members: *hosts
         
- !grant
  role: !group /conjur/authn-azure/AzureAE1/apps
  member: !group azure-apps
```
Step 5 : Define Conjur secrets and a group that has permissions on the secrets.
```ruby
  authn-azure-secrets.yml
  conjur policy load root authn-azure-secrets.yml
  =======================
- !policy
  id: secrets
  body:
    - !group consumers

    - !variable test-variable

    - !permit
      role: !group consumers
      privilege: [ read, execute ]
      resource: !variable test-variable

- !grant
  role: !group secrets/consumers
  member: !group azure-apps

```
Step 6 : Populate the secret with a value using the following CLI command
```ruby
  conjur variable values add secrets/test-variable m575ec2rtTest888wqertgawe2sadsfg6dsasdfa
````

Once I loaded the required properties listed above then the same can be verified via Conjur portal. I've put togather the slides in the below picture.
  <br>
![Eclipse](/assets/images/blogs/azauth/4portal.png)
<br>
## Implementation - Part2
Now I am going to explain how an application which is running on windows VM(Azure) will retrieve a secret(stored in a Conjur variable called : *secrets/test-variable*) without any hard coded tokens or passwords. For this demo I am going to use PowerShell for all the API calls because it easy for demo, but you can use any programming language as you like for ex: python, java etc.

1) An application requests its Azure AD token from the Azure Instance Metadata Service (IMDS). For this I am going call Microsoft’s non routable Public IP 169.254.169.254 which will provide metadata information which will be accessible only from running Azure virtual machines.

```ruby
$headers = New-Object "System.Collections.Generic.Dictionary[[String],[String]]"
$headers.Add("Metadata", "true")
$response = Invoke-RestMethod 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/' -Method 'GET' -Headers $headers                       
$access_token =$response.access_token
echo "$access_token"
```
2)  The IMDS responds with a signed JWT token. You can validate this token by any means. I've used https://jwt.io/ to decode the JWT and I'm able to validate my Azure Instance's Hostname, Object ID of System assigned Identity to make sure IMDS providing the correct details. <br>
![Eclipse](/assets/images/blogs/azauth/2aztoken.png)
<br>
3) Now the application sends an authentication request to Conjur using the Azure Authenticator REST API. In my lab I've only a standalone DAP instance runnning and I've going to send the REST request to DAP
```ruby

$headers1 = New-Object "System.Collections.Generic.Dictionary[[String],[String]]"
$headers1.Add("Content-Type", "application/x-www-form-urlencoded")
$headers1.Add("Accept-Encoding", "base64")
$body = "jwt=$access_token"
$response1 = Invoke-RestMethod 'https://dap-standalone.corpad.com/authn-azure/AzureAE1/dev/host%2Fazure-apps%2FAD/authenticate' -Method 'POST' -Headers $headers1 -Body $body
echo "$response1"
```
4) When Conjur authenticate and Authorize the request anf if it is successful, Conjur sends a short-lived access token back to the application.
<br>
![Eclipse](/assets/images/blogs/azauth/3conjurtoken.png)
<br>
5) Finally The application can get the secrets stored in Conjur using the access token I got from step 4.
The below simple Posershell commands will be used to make this REST call.


```ruby
$headers2 = New-Object "System.Collections.Generic.Dictionary[[String],[String]]"
$headers2.Add("Authorization", "Token token=`"$response1`"")
$response2 = Invoke-RestMethod 'https://dap-standalone.corpad.com/secrets/dev/variable/secrets/test-variable' -Method 'GET' -Headers $headers2

echo "The Retrieved Conjur secret is" :
echo "$response2" 
```
Finally Conjur realses the secrets as shown below:
<br>
![Eclipse](/assets/images/blogs/azauth/5secret.png)
<br>
The entire Powershell script is at the github and you can access them at [Nirmal's GitHub repo][jekyll-gh]. For any further queries. please reach out to me. Good luck!

[jekyll-gh]:  https://github.com/nirmal-dhanaraj/Azure-Authenticator

## Summary 

Conjur supports industry-standard authentication methods like Azure (IDMS), AWS (STS), GCP, OIDC LDAP etc. And I believe many more will be on the pipeline. In most of the cases Authentication and authorization to Conjur is based on credentials and automatically expiring access tokens. These ephemeral access tokens are required for all subsequent API requests after presenting credentials. They are cryptographically signed (RSA 2048) and expire after eight minutes. Surely, I recommend these features to my customers to increase and enhance the security posture of their AZURE workloads. 