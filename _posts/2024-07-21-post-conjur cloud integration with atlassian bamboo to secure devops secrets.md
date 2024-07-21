---
title: "Atlassian's Bamboo Integration with CyberArk's Conjur Cloud Secrets Manager"
comments: true
last_modified_at: 2024-07-20T07:07:02-05:00
show_date: true
related: true
share: true
categories:
  - Blog
tags:
  - Conjur Cloud
  - CyberArk
  - Atlassian Bamboo and Conjur Secrets Management
  - CI/CD & DevOps
  - Cyberark Identity Security Platform 
---

In this post, I'll guide you through the integration of Conjur Cloud with the Atlassian Bamboo CI/CD pipeline. This integration plays a crucial role in enhancing the security of Bamboo workloads by enabling them to securely retrieve credentials from Conjur Cloud. By leveraging Conjur's robust secrets management capabilities, Bamboo can effectively manage and protect sensitive information throughout the CI/CD process. I'll cover how to set up this integration, configure Bamboo to interact with Conjur Cloud securely, and highlight the benefits of centralizing secrets management for your CI/CD workflows.

<br>
![Eclipse](/assets/images/blogs/bamboo/bd.png)
<br><br>

## Prerequisites
- Secret managers plugin must be installed and enabled on the Bamboo administration Portal. 
- You must have active Conjur Cloud Tenant from CyberArk Identity Platform.
## Implementation - Part1
Step 1 : As the fitst step I'm uploading the below policy into Conjur Cloud to create a specific Policy structure which will be latter used to keep all the DevOps Workloads. You need to have Conjur admin role in Identity to load the below policy via the CLI tools. You can also automate this policy upload via REST APIs. 
```ruby
# conjur policy load -f C:\Tools\cc-cli-1.1.2\Conjur_Policies\Bamboo\branch-data.yaml -b data

- !policy
  id: apps
  owner: !group /Conjur_Cloud_Admins
  body:
    - !policy
      id: cicd-apps
```

Step 2 : Now, we need to create and define Workloads in Conjur Cloud that represent the actual DevOps workload. These Workloads will be used to authenticate as hosts and access secrets. Workloads can be created using various methods such as the UI or CLI. In this example, we'll demonstrate how to create a Bamboo DevOps workload via the Conjur Cloud Portal and how to grant permissions to Privilege Cloud Safes where the actual secrets are securly stored. <br> <br>
Step 2.1) Log in to Conjur Cloud Portal and go Resources and then Create new Workload via Workload builder window. I'll choose Workload type as "Other" for this Bamboo workload and press "Next" Button.
<br>
![Eclipse](/assets/images/blogs/bamboo/wl1.png)
<br><br>
Step 2.2) Now I need to provide a suitable name to the Workload "bamboo-pro1"  and I need to choose the location for the Workload in the Policy structure. As per the step1 above I'm choosing data/apps/cicd-apps as the location to keep this Workload and press "Next" Button.
<br>
![Eclipse](/assets/images/blogs/bamboo/wl2.png)
<br><br>
Step 2.3) 
<br>
![Eclipse](/assets/images/blogs/bamboo/wl3.png)
<br><br>
Step 2.4) 
<br>
![Eclipse](/assets/images/blogs/bamboo/wl4.png)
<br><br>
Step 2.5) 
<br>
![Eclipse](/assets/images/blogs/bamboo/wl5.png)
<br>
## Implementation - Part2
<b>Step 3.1) Bamboo Setup & Configure Plugin :</b>
Ensure that the Secret Managers for Bamboo plugin is successfully installed on your Bamboo server. Once installed, This will add a 'Secret Managers' menu item to the bottom of the Build Resources section as shown in the below picture.
And then select ‘CyberArk Conjur’ from the ‘Add New Manager’ drop-down and update the host details we have noted earlier as shown the below picture. Here you have the option to Test and validate the Workload connectivity between Bamboo and Conjur Cloud. We will save the details after the successful validations.
<br>
![Eclipse](/assets/images/blogs/bamboo/AB1.png)
<br><br>
<b>Step 3.2) Define Plan Secrets Variables :</b>
For this demo I have created a new Bamboo Plan with a Script Task where Variables can be used to make Secret values available when building plans in Bamboo. 
```ruby
Here we have to use the following format when referencing a variable: ${bamboo.variableName}
There are two Bmaboo variables refering in this demo which are DB_address & DB_password
```

<br>
![Eclipse](/assets/images/blogs/bamboo/AB2.png)
<br><br>
<b>Step 3.3) Map Plan variables with Conjur Variables :</b>
These Plan variables in Bamboo provide a powerful way to manage and customize build processes, making continuous integration and deployment more efficient and manageable. These Plan varibles will be used to map with Conjur Variables. In this demo, I have defined two Conjur Secrets as a Plan variable using a syntax below
```ruby
%<conjur:<conjur-secret-path>%
```
<br>
![Eclipse](/assets/images/blogs/bamboo/AB3.png)
<br><br>
<b>Step 3.4) Test and Validate :</b>
When executing this plan, the logs will reference the Secrets Resolver pre-build action, which dynamically resolves secrets via CyberArk Conjur mSecrets anager Platform. The actual secret values exist in memory only for the duration of the build or deployment. An important thing here is that this integration guarantees that secrets are never exposed in the logs
<br>
![Eclipse](/assets/images/blogs/bamboo/AB4.png)
<br> <br>
<b>Step 3.5) Audit Trails :</b>
The graphics below depict CyberArk's unified audit and monitoring service for all hosted CyberArk business services, including Conjur Cloud. <br>
![Eclipse](/assets/images/blogs/bamboo/CC-FS1.png)

## Summary 

Discover how to fortify your Atlassian Bamboo CI/CD pipelines with robust secrets management practices using CyberArk Conjur. By leveraging external secret management platforms, such as CyberArk Conjur, you can shield sensitive information from exposure during builds and deployments. Utilize a streamlined syntax to securely reference external secrets across various Bamboo variables, safeguarding your project, and environment configurations.

Integrating this approach into Bamboo Specs ensures that credentials remain protected and never exposed in source control. Secrets are dynamically resolved within Conjur Cloud, ensuring they exist in memory only for the duration required during builds or deployments. This method guarantees that secret values are consistently obfuscated in logs, bolstering your pipeline's overall security posture.

Explore these strategies to elevate your Bamboo workflows with advanced secrets management and robust security practices.