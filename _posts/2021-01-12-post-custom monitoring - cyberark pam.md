---
title: "Custom Monitoring - CyberArk PAM"
comments: true
last_modified_at: 2021-01-12T08:20:02-05:00
show_date: true
related: true
share: true
categories:
  - Blog
tags:
  - CyberArk
  - REST API
  - PAM
  - Monitoring
---

 <br>
![Eclipse](/assets/images/blogs/monitoring/design.png)
<br><br>

Service Monitoring plays a vital role in managing any enterprise applications effectively. Most of the CyberArk products supports effective monitoring capabilities through SIEM integrations. Generally, SIEM systems are integrated with ITMS (ex: ServiceNow)  solutions to effectively maintain Service Level Agreements and other tracing purposes.

In my recent engagement with one of the customers who is in the energy business, during the initial analysis phase I came to know that the customer doesn’t have any SIEM solutions or SNMP based monitoring solutions however they are in the process of implementing Splunk in the near future.   

Monitoring CyberArk Password Vault and component services are very crucial for successful PAM deployment. 
I’ve proposed a java spring-boot based custom monitoring solution which utilizes PAM REST capabilities to monitor all the PAM Complements as well as it will create a ServiceNow Incident when a particular PAM component is not accessible ( Vault, PVWA, PSM(P),CCP &PTA ).
In this custom solution Im going to use two different PAM REST Services and one ServiceNow REST Service 

1. CyberArk System health details : This will return the details about components specific details,  system health information for each instance.
2. PAS system health summary : This will return the consolidated information about the Vault, PVWA, CPM, PSM/PSM for SSH, PTA, and CCP.
3. create an incident record : This ServiceNow REST Service will be used for creating an Incident through REST clients.

Note : In the past I’ve written few blog posts relating to CyberArk REST capabilities and its worth going through them for better understanding 
Pre-Requisites: <br><br>

•	 JDK 1.8 or more ( I’m using JDK 14)<br>
•	Eclipse (IDE for java & Spring boot)<br>
•	Maven (Build tool)<br>
•	CyberArk Service Account for initials authentications <br>
•	ServiceNow Service Account with appropriate permissions to create INC records<br>
•	If you wanted to enable SSL between your java environment and PVWA you need to import appropriate certificate into java key store( which Im not covering here)<br><br>

For simplicity, Im going to deal with only three CyberArk PAM REST services in this which are as follows <br>
-	AuthServices  :  Im invoking this service primarily for getting the CyberArk PAM authorization tokens <br>
-	GetSystemHealthSummaryServices : In this REST call I’m going to use the above authorization token to get list of all components ID’s  <br>
-	GetSystemHealthServices : In this REST call I’m going to use the above authorization tokens as well as components ID’s  to get the status of each instance of the CyberArk Services and if needed I’ll create a ServiceNow INC.<br>
-	Logout : Finally, Im going to invoke the logout REST services to nicely invalidate the tokens.<br>
 

In this post I’m going to demonstrate how we can leverage GetSafes rest services using java spring-boot based REST API calls. And this project can be used as a template for addition service inclusions for PAM automations.


I’ve created a java spring boot project in Eclipse to implement the above client services . 

Traditionally java main method resides inside the GetSafeMembersApplication java class where the executions starts 
```ruby
package com.pam.monitoring;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class CustomMonitoringApplication {

	public static void main(String[] args) {
		SpringApplication.run(CustomMonitoringApplication.class, args);
	}

}
```
<br>
Another important class is called as LoginController java which will be exposing two local endpoints, one for invoking the authorization token and the other one for get list of all Safes
<br>
```ruby
package com.pam.monitoring.controller;

import java.io.IOException;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

import com.pam.monitoring.service.AuthService;
import com.pam.monitoring.service.GetSystemHealthService;
import com.pam.monitoring.service.GetSystemHealthSummaryService;

@Controller
public class LoginController {
	
private final AuthService authService;
private final GetSystemHealthSummaryService getSystemHealthSummaryService;
private GetSystemHealthService getSystemHealthService;
	
	@Autowired
	LoginController(final AuthService authService, final GetSystemHealthService getSystemHealthService, final GetSystemHealthSummaryService getSystemHealthSummaryService){
		this.authService= authService;
		this.getSystemHealthSummaryService=getSystemHealthSummaryService;
		this.getSystemHealthService=getSystemHealthService;
	}
	
	@RequestMapping("/getAccessToken")
	public ResponseEntity<String> getAccessToken() throws Exception {
		return authService.getAccessToken();
	}
	/*
	 * @RequestMapping("/getHealthMonitoring") public ResponseEntity<String>
	 * getSystemHealth(@RequestParam String compId) throws Exception { return
	 * getSystemHealthService.getSystemHealth(compId); }
	 */
	
	@RequestMapping("/getMonitoringSummary")
	public ResponseEntity<String> getSystemHealthSummary() throws Exception {
		return getSystemHealthSummaryService.getSystemHealthSummary();
	}

}
```
This custom solutions has been scheduled with the help of java based scheduler and it is implemented in HealthScheduler.java. Therefore we need not plan for any OS based schedulers.
```ruby
package com.pam.monitoring.component;

import java.io.IOException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import com.pam.monitoring.service.GetSystemHealthSummaryService;

import lombok.extern.slf4j.Slf4j;
@Slf4j
@Component
public class HealthScheduler {

	@Autowired
	private GetSystemHealthSummaryService getSystemHealthSummaryService;

	@Scheduled(fixedDelay = 5000) // This value in milliseconds and we can adjust the frequency as per your need
	public void scheduleHealthSummary() {
		try {
			System.out.println("========== ============= ");
			log.info(" Health Check Starting ");
			ResponseEntity<String> response = getSystemHealthSummaryService.getSystemHealthSummary();
			System.out.println("Response -----------------"+response);
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
```
When every thing goes well ,you can run spring boot applications and if there is no error it will automatically retrieve the status of all the instances of each component and If it finds 
any component id is down, then this custom solution will create a ServiceNow Incident record. 
<br>
<br>
To demonstrate this solution, Im manually turning off the PSM Service through CyberArk PVWA web portal as given below:
 <br>
 ![](/assets/images/blogs/monitoring/SysHealth.png)
 <br> <br>
 This custum solution detect PSM Component inactivity and it will create an Incident record in my Dev ServiceNow instance as shown below.<br>
 <br>
 ![](/assets/images/blogs/monitoring/snowinc.png)
<br>
The entire project is at the github and you can access them at [Nirmal's GitHub repo][jekyll-gh]. you can import into your local IDE for your own usage. 
If you have any further queries please reach out to me. 

[jekyll-gh]:  https://github.com/nirmal-dhanaraj/cyberark_pam_monitoring
## Summary 
I’ve prepared this for my personal CyberArk PAM monitoring and administration purpose. I’m sharing here for anyone who wanted to automate their CyberArk PAM solutions. This project can be used as a modal/template and you can build more services like sending emails when a compoenent is down etc.. If you have any further queries feel free get in touch and Im happy to help.