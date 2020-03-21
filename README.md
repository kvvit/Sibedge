# Test task answers
### 1.Please write iptables rules that will block all incoming traffic and allow outgoing (browsing from local machine should work, but all incoming connections should be dropped).
Answer: most easy way to set policy for chains INPUT and FORWARD to DROP (or REJECT) and for chain OUTPUT to ACCEPT. I also usually open all for loopback interface. File from iptables-restore take rules has view:
```
*filter
-A INPUT -i lo -j ACCEPT
-A OUTPUT -o lo -j ACCEPT

-A INPUT -i enp0s3 -m state --state ESTABLISHED,RELATED -j ACCEPT

-P INPUT DROP
-P FORWARD DROP
-P OUTPUT ACCEPT

COMMIT
```
But I usually set policy for all chains to DROP, and my config file for iptables has view:
```
*filter

-A INPUT -i lo -j ACCEPT
-A OUTPUT -o lo -j ACCEPT

-A INPUT -i enp0s3 -m state --state ESTABLISHED,RELATED -j ACCEPT
-A OUTPUT -o enp0s3 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

-A INPUT -i enp0s3 -s 192.168.1.101/32 -p tcp -m tcp --dport 22 -m state --state NEW -j ACCEPT
-A INPUT -i enp0s3 -s 192.168.1.101/32  -p icmp -m icmp --icmp-type 8 -j ACCEPT

-A INPUT -j LOG

-P INPUT DROP
-P FORWARD DROP
-P OUTPUT DROP

COMMIT
```
Here is enp0s3 ethernet interface in local machine (can be different in some virtuallization systems), and also I open ssh and icmp for my management station (it ip-address is 192.168.1.101)
### 2. There is a directory with 2 files that are shown as '???' and '???'. They have different names, but in console are shown as 3 question marks. How to remove them?
Answer: I haven't encountered with this situation, probably it can be due to problems with encoding. If in directory only this two files, the most easy way to delete them is next command:
```bash
rm -rf /path_to_dir/*
```
It command usualy remove all content of directory.
Another solution for this situation, can be remove files by it's inode. I can't make it with one string command (with pipe) and write this little script:
```bash
#!/bin/bash
PATH_PROBLEM=/path_to_problem_files/
inodes=`ls -li $PATH_PROBLEM | awk '{print $1}' | awk 'NR>1'`
for inode in $inodes
do
  find . -inum $inode  -exec rm {} \;
done
```
### 3. If you want to upload some file once per hour to 1 server from 1000 servers, how it can be done? What special cases should be taken into account?
Answer: We admit, that we can connect to 1000 servers via ssh, and  that we use rsa-key. In my examples I chose nginx.conf file, and check if it exist on remote server. For the name of copied file I use next format: nginx.conf-remote_server_name-timestamp (to avoid duplicate names). I would like to show two ways how to solve our task (this two methods use ssh):
- Use bash script. I have written file, with list of names of our servers, it called servers.txt, and have view:
```
[all]
server0001.somedomainname
server0002.somedomainname
----
server1000.somedomainname
```
And bash script, that use list of servers (in same directory with servers list file)
```bash
#!/bin/bash
FILE_PATH=/etc/nginx/nginx.conf
SCRIPT_PATH=/path_to_script/
servers=`cat ${SCRIPT_PATH}servers.txt |  awk 'NR>1'`
for s in $servers
do
  if ssh $s stat $FILE_PATH \> /dev/null 2\>\&1
    then
      echo "File exists on $s"
      timestamp=`date +"%H%M%S%d%m%Y"`
      scp $s:$FILE_PATH ${SCRIPT_PATH}nginx.conf-$s-$timestamp
    else
      echo "File does not exist on $s"
  fi
done
```
After that, I have made cron task in /etc/crontab (or can create file with same content in /etc/cron.d) with next content:
```
0 */1 * * * user bash /path_to_script/script.sh
```
- Use ansible. For inventory I have used same file with servers list, and have written next playbook (with check if remote file exist):
```yml
---
- hosts: all
  become: no
  tasks:

  - name: Get file from remote location
    fetch:
      src: /etc/nginx/nginx.conf
      dest: ./nginx.conf-{{ inventory_hostname }}-{{ansible_date_time.iso8601_basic}}
      fail_on_missing: no
      flat: yes
```
As in first example I have made cron task in /etc/crontab with next content:
```
0 */1 * * * user ansible-playbook -i /path_to_script/servers.txt /path_to_script/files.yml
```
### 4. You have a folder that stores backups of a few projects. Each project lays in a separate folder and each backup is a tarball file with unique name. Write a shell script that removes all backups for the given project except the last 5.
Answer: I assume, that backup directory is /backup. In that directory there are another directories with backup tarball files. In that directories can be placed other type of files, and my script don’t affect on this files. Below is contain of the script for our task:
```bash
#!/bin/bash
BACKUP_PATH='/backup/'
cd ${BACKUP_PATH}
DIRS=`ls -d */`
for DIR in $DIRS
do
  cd $BACKUP_PATH$DIR
  ls *.tar -t 2> /dev/null | awk 'NR>5' | xargs rm 2> /dev/null
done
```
