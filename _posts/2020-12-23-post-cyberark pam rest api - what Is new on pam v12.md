---
title: "CyberArk PAM REST API - What Is New on PAM V12"
comments: true
last_modified_at: 2020-12-23T16:20:02-05:00
show_date: true
related: true
share: true
categories:
  - Blog
tags:
  - CyberArk
  - REST API
  - PAM
---

 <br>
![Eclipse](/assets/images/blogs/pamv12/architecture.png)
<br><br>

CyberArk has recently announced Privileged Access Security solutions version 12 and in this Post I’d like to analyze the cool features around its robust REST web services capabilities..
Few days ago CyberArk anounced their new version of PAM solutions and I've seen some notable new REST APIs such as:

Get Safes  -  This service returns list of all Safes that the requested user has permissions to view. This API has come up with several features such as paging and searching based some   			     parameter-based filters.
Get members -  Returns a list of all the members of a specific Safe.
Get Users & Get groups - also has come up with some enhancements. 

In this post I’m going to demonstrate how we can leverage GetSafes rest services using java spring-boot based REST API calls. And this project can be used as a template for addition service inclusions for PAM automations.


Pre-Requisites:
Make sure your workstation/ Laptop is installed with the below tools for this simple exercise.<br><br>
•	JDK 1.8 or more <br>
•	Eclipse (IDE for java & Spring boot)<br>
•	Maven (Build tool)<br>
•	CyberArk Service Account for initials authentications <br>
•	If you wanted to enable SSL between your java environment and PVWA .. you need to import appropriate certificate into
    java key store(which Im not covering here)<br>
<br>
To maintain simplicity Im going to deal with only three CyberArk PAM REST services here which are: <br>
-	Login : Im invoking this service primarily for getting the CyberArk PAM authorizations tokens <br>
-	GetSafes : In this REST call I’m going to use the above authorization tokens to get the list of Safes <br>
-	Logout : Finally, I need to invoke the logout rest service to invalidate the token.<br><br>

I’ve created a java spring boot project in Eclipse to implement the above client services . 

Traditionally java main method resides inside the GetSafeMembersApplication java class where the executions starts 
```ruby
package com.pam.v12;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GetSafeMembersApplication {

	public static void main(String[] args) {
		SpringApplication.run(GetSafeMembersApplication.class, args);
	}
}

```
<br>
Another important class is called as LoginController java which will be exposing two local endpoints, one for invoking the authorization token and the other one for get list of all Safes
<br>
```ruby
package com.pam.v12.controller;

import java.io.IOException;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

import com.pam.v12.service.AuthService;
import com.pam.v12.service.GetSafeService;

@Controller
public class LoginController {
	
private final AuthService authService;
private final GetSafeService getSafeService;
	
	@Autowired
	LoginController(final AuthService authService, final GetSafeService getSafeService){
		this.authService= authService;
		this.getSafeService= getSafeService;
	}
	
	@RequestMapping("/getAccessToken")
	public ResponseEntity<String> getAccessToken() throws Exception {
		return authService.getAccessToken();
	}
	
	@RequestMapping("/getSafeLists")
	public ResponseEntity<String> getSafeLists() throws Exception {
		return getSafeService.getSafeLists();
	}

}
```
<br>
I’ve two service classes which are AuthService.java and GetSafesService.java. These classes will used here to handle the actual business logics related to Request and Response for the above two mentioned services 
```ruby

package com.pam.v12.service;
import java.io.IOException;
import java.net.URI;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.util.Collections;
import java.nio.charset.StandardCharsets;
import java.util.Base64;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

import com.pam.v12.model.AuthRequest;
import com.pam.v12.util.AuthRequestBuilder;
import com.pam.v12.util.AuthUtil;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
public class GetSafeService {
	@Value("${application.logon.url}")
	private String access_token_url;
	@Value("${application.logout.url}")
	private String logOff_url;
	@Value("${auth.username}")
	private String username;
	@Value("${auth.pass}")
	private String password;
	
	@Autowired
	private AuthService authService;
	
	public ResponseEntity<String> getSafeLists() throws IOException
	{		
		String authorizationToken1= AuthUtil.authlogin();		
		
		RestTemplate restTemplate = new RestTemplate();

	    HttpHeaders headers1 = new HttpHeaders();
	    String authorizationToken=authorizationToken1.substring(1, authorizationToken1.length()-1);
	    System.out.println("Token : "+authorizationToken); 
	    
	    //adding the query params to the URL
        UriComponentsBuilder uriBuilder = UriComponentsBuilder.fromHttpUrl("http://compsrv1.corpad.com/PasswordVault/api/Safes")
                .queryParam("includeAccounts", "true");
        
        
	    headers1.setAccept(Collections.singletonList(MediaType.APPLICATION_JSON));
	    headers1.setContentType(MediaType.APPLICATION_JSON);
	    headers1.add("User-Agent", "Spring's RestTemplate" ); 
        headers1.add("Authorization", authorizationToken);
	    
		HttpEntity<String> requestEntity = new HttpEntity<String>(null,headers1);
	    
		ResponseEntity<String> responseEntity = restTemplate.exchange(
                uriBuilder.toUriString(),
                HttpMethod.GET,
                requestEntity,
                String.class
        );		
		
		System.out.println("Response --------------------------"+responseEntity.getBody()); 
		
		ResponseEntity<String> logoutResponse = authService.logoutAccessToken(authorizationToken1);
	    System.out.println("Logout Response : "+logoutResponse.getBody().toString()); 
	    return responseEntity;

	}
	
}

```
<br>
<br>
In general use case once we get the  authorization token which then will be used for subsequent operation like safe creation, modification etc. when the required operation is completed, we need to logout properly.
<br>
 Class AuthService.Java is main service calss where spring boot constructs the request module and post REST request to
 CyberArk PVWA.
 <br><br>
 For simplicity the REST service is invoked via browser and you can also use Postman too to invoke the same.<br>
 <br>
![Eclipse](/assets/images/blogs/pamv12/response.png)
<br>
If there is no error you will get the authorization and that will be passed as an Header to another services to get the list of all Safes when you execute/run the code through Eclipse ..
<br>
![Eclipse](/assets/images/blogs/pamv12/result.png)
<br>

The entire project is at the github and you can access them at [Nirmal's GitHub repo][jekyll-gh]. you can import into your local IDE for your own usage. 
If you hve any further queries please reach out to me. 

[jekyll-gh]:  https://github.com/nirmal-dhanaraj/cyberark_pam_rest_v12

## Summary 
I’ve prepared this for my personal CyberArk automations and administration purpose. I’m sharing here for anyone who wanted to automate their CyberArk PAM solutions. This project can be used as a modal/template and you can build more services based on your requirements on top of it. If you have any queries feel free get in touch and happy to help