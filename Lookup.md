Room Name : Lookup

Difficulty : Easy

Room Link : https://tryhackme.com/room/lookup

<br><br>

## TL:DR

I owned the box by exploiting an exposed elFinder instance to get a web shell as www-data, used a PATH hijack against a vulnerable SUID binary (/usr/sbin/pwm) to recover credentials for user think, SSH'd into the box as think, abused a sudo rule allowing look to read /root/.ssh/id_rsa, and finally used that private key to SSH as root.

<br> 

## 1) Foot in the door — reconnaissance

I started with a standard nmap scan to see which services were running
<br><br>

<img width="938" height="204" alt="Screenshot 2025-09-14 232056" src="https://github.com/user-attachments/assets/df0d0aaf-6842-4e8a-a5c6-d80239d9dc7c" />
<br><br>

Findings: SSH (22) and HTTP (80) were open. I added lookup.thm to /etc/hosts and visited the site in a browser — a simple login page appeared.
<br><br>

<img width="450" height="311" alt="image" src="https://github.com/user-attachments/assets/e5b91f2d-9899-4b75-941c-b703a1a72bc5" />
<br><br>

By poking the login form, I noticed different responses for “unknown user” vs “wrong password” — enough to enumerate valid usernames.
<br>
<br>


## 2) Enumeration — find the file manager

I tried some simple credentials like "admin : admin", but it didn't work.
I Observed it is possible to enumerate valid usernames based on the errors we are getting on the website upon a failed login.


<br><br>
<img width="321" height="184" alt="image" src="https://github.com/user-attachments/assets/a74c4f4a-a032-4c10-8521-abb824b825e5" />

<img width="408" height="118" alt="image" src="https://github.com/user-attachments/assets/eddd45a9-4d75-41e3-b8de-a9a781d38501" />

<br><br>

We can write a small python script to enumerate usernames 
<br><br>
```

import requests

# Define the target URL
url = "http://lookup.thm/login.php"

# Define the file path containing usernames
file_path = "/usr/share/seclists/Usernames/Names/names.txt"

# Read the file and process each line
try:
    with open(file_path, "r") as file:
        for line in file:
            username = line.strip()
            if not username:
                continue  # Skip empty lines
            
            # Prepare the POST data
            data = {
                "username": username,
                "password": "password"  # Fixed password for testing
            }

            # Send the POST request
            response = requests.post(url, data=data)
            
            # Check the response content
            if "Wrong password" in response.text:
                print(f"Username found: {username}")
            elif "wrong username" in response.text:
                continue  # Silent continuation for wrong usernames
except FileNotFoundError:
    print(f"Error: The file {file_path} does not exist.")
except requests.RequestException as e:
    print(f"Error: An HTTP request error occurred: {e}")

```

<br>

Make sure to change the path of file which contains the names. change it to the path where your names.txt is located.

And by running this script we find another username, "jose".

Let's use Hydra to perform brute force attack for "jose".
<br>
<br>

```
hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong" -V
```
<br><br>
After a successfull brute force attack we get. 
<br>

<br>

<img width="791" height="64" alt="Screenshot 2025-09-15 024553" src="https://github.com/user-attachments/assets/f36fdc76-a3de-4ee6-b9a7-10a81ae4040c" />

<br>
<br>
And here we go! 
<br>
<br>

## 3) Discovering ElFinder

After logging in with the credentials we first get an error, and get redirected to "files.lookup.thm".

Let's add that to our hosts file and check it out.

<br>
<img width="842" height="383" alt="Screenshot 2025-09-15 024745" src="https://github.com/user-attachments/assets/52ffafde-d74a-4da2-b9a6-63fb01d5efae" />
<br><br>

Now we Land in 'elfinder', Looks like a file manager.

<br>

<img width="1290" height="520" alt="Screenshot 2025-09-15 024940" src="https://github.com/user-attachments/assets/f13ef49c-a1a4-49af-9343-e2f4b534116c" />
<br>
<br>
There are quite few files here, and they seem to contain passwords. Let's Find out what the version of elfinder is.
<br>
<br>
<img width="544" height="415" alt="Screenshot 2025-09-15 025023" src="https://github.com/user-attachments/assets/9bf84cfe-135c-425c-b954-f1b9722db984" />
<br>
We successfully found the version of elfinder. it is 2.1.47.

Searching for known exploits for elfinder, we can see a metasploit module that targets versions before 2.1.48

<br>
<img width="861" height="169" alt="Screenshot 2025-09-15 025117" src="https://github.com/user-attachments/assets/39a68f65-7894-41fb-9b33-b8f2a1cbeb1f" />
<br>
<br>
<img width="677" height="190" alt="image" src="https://github.com/user-attachments/assets/263b1dcf-3dd8-4271-9a8a-219237140130" />
<br>
<br>

we set RHOST option to 'files.lookup.thm' and LHOST to our kali machines IP.

<br><br>

## 4) We get First shell as www-data

and we get shell as 'www-data'

<img width="488" height="67" alt="Screenshot 2025-09-15 030734" src="https://github.com/user-attachments/assets/e5c4435c-e573-4fc2-9ba6-141de97824f0" />
<br>
<br>
<img width="503" height="77" alt="Screenshot 2025-09-15 031635" src="https://github.com/user-attachments/assets/c7c61129-f35a-49e6-bf36-727eaf8125db" />
<br><br>
now lets look into etc/passwd file, we find out that there are users named, root and think.
<br><br>
<img width="512" height="81" alt="Screenshot 2025-09-15 032101" src="https://github.com/user-attachments/assets/c7e1b131-9a39-482b-93f2-3162d3accb51" />
<br><br>
the user 'think' had password file in home directory. But, we didn't have permission to read it. 

Let's find SUID binaries.
<br><br>

```
find / -perm /4000 2>/dev/null
```
<br>

<img width="385" height="151" alt="Screenshot 2025-09-15 032209" src="https://github.com/user-attachments/assets/33d30e0a-de8e-4e82-877a-4690c04efeff" />
<br><br>

The /usr/sbin/pwm binary draws my attention, because its not there by default on Linux hosts.

Let's see what happens when we execute it.

<br><br>
<img width="612" height="81" alt="Screenshot 2025-09-15 032846" src="https://github.com/user-attachments/assets/9fcb066c-b4b5-4a50-abef-aebdf1870df8" />
<br><br>

This binary seems to execute 'id' command and then extracts username out of it, and puts that username into “/home/<username>/.passwords”.
<br><br>

## 5) Abusing pwm with PATH hijack

If the “id” command is not specified with it’s full path (/bin/id), it is found and executed via the PATH variable in our environment.
<br><br>
<img width="600" height="48" alt="Screenshot 2025-09-15 032935" src="https://github.com/user-attachments/assets/98ae7a14-734f-44b6-b317-7d1b7db32061" />
<br><br>
Let’s try to trick the program into executing a different “id” command, so that it would result in the “think” username to be extracted from the output.

let's add /tmp to our path and create /tmp/id with the following content.

<br><br>
<img width="687" height="123" alt="Screenshot 2025-09-15 034017" src="https://github.com/user-attachments/assets/857b8b86-158e-4fd3-9baa-dfa0b9a6f4fc" />

<br><br>
<img width="480" height="47" alt="Screenshot 2025-09-15 033706" src="https://github.com/user-attachments/assets/900e124a-5013-4931-81f8-c716a047269b" />
<br><br>

Let's try running "pwm" again 
<br><br>

<img width="672" height="360" alt="Screenshot 2025-09-15 034215" src="https://github.com/user-attachments/assets/b4dd0e52-283e-46c8-9ca9-1d8b698af76d" />
<br><br>

And voila! we got password list by tricking binary into extracting 'think' as username.

Let's save this list and brute force 'think'
<br><br>

## 6) SSH as think

Let's run hydra command to obtain correct credentials.

<br>

```
 hydra -l think -P /home/mystic/Downloads/test/htb/password.txt  -t 4 ssh://<IP>
```

<br>

and we get the credentials!
<br><br>
<img width="700" height="57" alt="Screenshot 2025-09-15 035904" src="https://github.com/user-attachments/assets/af7ae30b-af19-4cbb-af35-1d0b983f87d9" />
<br><br>

Now we can login via SSH.
<br><br>
<img width="604" height="62" alt="Screenshot 2025-09-15 040148" src="https://github.com/user-attachments/assets/d1c5952f-258b-46d5-acd0-22a7a9fb401e" />
<br><br>

## Escalation to root

Now let's check 'think' privileges

we find out 'think' can use 'look' command as root.

<br><br>
<img width="868" height="133" alt="Screenshot 2025-09-15 040559" src="https://github.com/user-attachments/assets/c3cb0d92-109b-45e9-bd16-f8ce145fbed4" />
<br><br>

Let's check GTFOBins to see if we can exploit this.

<br><br>
<img width="680" height="134" alt="image" src="https://github.com/user-attachments/assets/a0f9b4ce-bae7-4b3b-ac46-0d9e57db81f0" />
<br><br>

According to GTFOBins, we can use this binary to read files on the system, and since we can run this as root, we can read any file on the system!

The fastest win would be root’s private SSH key

<br><br>
<img width="732" height="70" alt="Screenshot 2025-09-15 040712" src="https://github.com/user-attachments/assets/fd083307-9856-455c-98ca-139bd58e60d1" />
<br><br>

Let's save this key on our machine and login as root.

<br><br>
<img width="1108" height="440" alt="Screenshot 2025-09-15 042830" src="https://github.com/user-attachments/assets/9e6d3a98-e38d-4031-ba56-c8e2944e763e" />
<br><br>

And here we have it!! lookup has been R00T3D.
