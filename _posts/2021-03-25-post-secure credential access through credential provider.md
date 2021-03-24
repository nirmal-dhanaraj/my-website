---
title: "Secure Credential Access through Credential Provider"
comments: true
last_modified_at: 2021-03-25T07:07:02-05:00
show_date: true
related: true
share: true
categories:
  - Blog
tags:
  - CyberArk
  - Credential Provider
  - PAM
  - CyberArk Automation
---

 <br>
![Eclipse](/assets/images/blogs/cp/cparchi.png)
<br><br>
Securing hardcoded credentials has always been challenging for applications and platforms owners. Still organizations are looking for ways to protect privileged credentials spread across many different places like config file, scripts and schedule jobs etc.

Recently I've come across a scenario where one of my customers infrastructure monitoring team using plain text username and password in their monitoring scripts that are presents in the most of their windows servers. In this post Im going to demonstrate how we can use CyberArk Credential Provider to securely retrieve the required Credential through SDKs and Command line commands and this will provide  on-demand credentials for the scripts also it will adhere  with internal and external compliance requirements of periodic password change, and monitor privileged access across all platforms.

## Requirements
This is an agent based solution and this is recommended for business critical applications that require the highest level of availability, performance and security. secure local cache

- Install CyberArk Credential Provider (CP) Agent on the server : this is a light wait agent and it is available almost for all windows and Linux flavors
- CyberArk Provides SDK for Java and .Net and also supports CMD access : In this post Im going to demonstrate Privileged credentials access through Command line control as well as      through   Java SDKs
- Create an Application via PVWA Web Portal : we can enforce more restrictions at the application level. In this example Im using OSUser 
- Onboard Privilege Accounts and assign required permissions to the application as well as Application Provider User.
- Execute the command from the server or access credentials via java program


## Steps

1) Download and install CP agent from CyberArk SFE Portal to the server when we need to access the credentials. Make sure you open port 1858 accessible between the agent server and the Password Vault. After successful agent installation, the CyberArk administrator can validate through PVWA portal's System health page as shown below.

![Eclipse](/assets/images/blogs/cp/pvwa.png)

2)  for this demo I've Created a simple Application and enabled OSUser authentication (corpad\ndhanaraj) and configured IP Address Restriction. Meaning the password will be released to the server I mentioned and for the user corpad\ndhanaraj. Password cannot be accessed by other users other than the configured user(s) list.

![Eclipse](/assets/images/blogs/cp/app.png)
In simple terms, the whole PAS Orchestrator can be deployed in a single server (Unix/Linux or Windows) and this will take care your entire PAS components on the target windows machines. 

3) For this demo I've onboarded an Active Directory account into Password Vault and it is completely managed by CyberArk. I've provided required access to application I've created in the step 1 and retrieve and list access to provider user as shown in the below screenshot


![Eclipse](/assets/images/blogs/cp/safe.png)

4) Now I need to login to the windows server using my AD Account (corpad\ndhanaraj) and try to access the password through Command Line Command as shown below and you can Im able to retrieve the credentails from Password vault nicely !!
```ruby
  CLIPasswordSDK.exe GetPassword /p AppDescs.AppID=WIN-DOM-JavaApp /p Query="Safe=WIN-DOM-SVC;Folder=Root;Object=svc_mon_acc" /o Password
```
![Eclipse](/assets/images/blogs/cp/cmd.png)
5) In this step Im going to show you that how Im retrieving the required credentials through java. I've installed JDK on that machine and created a simple java project which will have two java classes and jar file (javaPassswordSDK.jar) (Note : there is JDK available for .Net as well)

RetrieveCredentialUsingSDK.java
```ruby
package pam.cp;

public class RetrieveCredentialUsingSDK {
	private static final String String = null;
	public static void main(String[] args) {
		// TODO Auto-generated method stub
        String appID="WIN-DOM-JavaApp";
        String safe="WIN-DOM-SVC";
        String svcAccountName="svc_mon_acc";
        String reason = "Billing application– connect to DB";
       	JavaCredentialSDK sdk =new JavaCredentialSDK();
		String result =  sdk.CredentialSDK(appID, safe, svcAccountName, reason);
		System.out.println();
		System.out.println("Password Access via Java SDK "+result);
	}
}
```
JavaCredentialSDK.java
```ruby
package pam.cp;

import java.util.Arrays;
import javapasswordsdk.*;
import javapasswordsdk.exceptions.*;

public class JavaCredentialSDK {
 
	public JavaCredentialSDK() {
		// TODO Auto-generated constructor stub
	}
	public String CredentialSDK(String AppID, String Safe,String svcAccountName,String reason)
    {
        PSDKPassword password = null;
        String cred = null;
        char[] content = null;
        try {
            PSDKPasswordRequest passRequest = new PSDKPasswordRequest();
            // Set request properties
            passRequest.setAppID(AppID);
            passRequest.setSafe(Safe);
            passRequest.setFolder("root");
            passRequest.setObject(svcAccountName);
            passRequest.setReason(reason);
         
            // Get password object
            password = javapasswordsdk.PasswordSDK.getPassword(passRequest);

            // Get password content
            content = password.getSecureContent();
            cred=String.valueOf(content);
        } catch (PSDKException ex) {
            ex.printStackTrace();
        } finally {
          if(content != null) {
            // Clean the returned object
            Arrays.fill(content, (char) 0);
          }
          if(password != null) {
              // Dispose of resources used by this PSDKPassword object
              try {
            	password.dispose();
				  
		        	} catch (PSDKException e) {
				      // TODO Auto-generated catch block
				          e.printStackTrace();
			        }
          }
        }
		return cred;
    } 
    }

```
5) Finally I got the required privileged credentials when Im running my java code as shown in the below screen shot
![Eclipse](/assets/images/blogs/cp/java.png)
## Summary 
I’ve deployed this in a windows 2019 server and I've used latest PAS version. This is an agent based solution and this is recommended for business critical applications that require the highest level of availability, performance and security. One other thing is that these agent based solutions are well suited for large organizations which are using distributed password vaults architecture since Satellite vault supports Credential providers.

The Credential Provider enables the highest level of anti-tampering security mechanism, as well as high-availability and performance, using the secure local cache. Furthermore, the local cache enables high performance password requests and high-availability in the case of a network outage to the Password Vault.