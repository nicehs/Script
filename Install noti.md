## Install Noti service

### 1. Install Tools
```
yum install borgbackup inotify-tools mailx
```
**More information:**
**borgbackup:** [General — Borg - Deduplicating Archiver 1.1.17 documentation](https://borgbackup.readthedocs.io/en/stable/usage/general.html)
**inotify-tools:** [Home · inotify-tools/inotify-tools Wiki · GitHub](https://github.com/inotify-tools/inotify-tools/wiki)
**mailx:** [mailx - Wikipedia](https://en.wikipedia.org/wiki/Mailx)
### 2. Run php_allowed.sh
```
#!/bin/bash

web_user=nobody
acl_file=/tmp/acl
docroots=( /path/to/doc_root )

for docroot in "$docroots"; do
    len=${#docroot}
	echo $len
	for file in $( find ${docroot}/ -type f ! -user $web_user -name "*.php" )
    do
        echo "${file}" >> $acl_file
    done
done
```
### 3. Create your first repo
```
mkdir /path/to/backup_file
borg init --encryption=repokey /path/to/backup_file
```
### 4. Backup root directory of your vhost
```
borg create /path/to/backup_file::"version 1" /full_path/to/root_directory
```
### 5. Touch inotify.sh
**You need to change argument below**
```
#!/bin/bash
#Passphrase need to secure borg when create a new version, extract, list, ...
#Put your passphrase here
export BORG_PASSPHRASE=your_passphrase

#You need to change these folders
docroots=/path/to/root_directory
repo=/path/to/backup_directory

#Config mailx
$smtp_server=your_server
$display_name=you_can_write_anything_here
$smtp_username=your_username
$smtp_pass=your_pass

#Watch folder
inotifywait -mr $docroots -e create,modify,delete,moved_from,moved_to |    
while read path action file; do

#Set number repo version
a="`borg list $repo --last 1 | cut -d " " -f2`"
dir_restore_file="`echo "$path" | sed 's/^.//'`"
strip_num="`echo "$dir_restore_file" | grep -o "/" | wc -l`"

#Restore a file from nearest repo which contain the lastest version of that file
function extract()
{
        if [ $a -gt 0 ]; then
                borg list $repo::"version $a"  | grep $dir_restore_file$file > /dev/null 
                if [ "$?" -eq 0 ]; then
                        borg extract $repo::"version $a" $dir_restore_file$file --strip-components $strip_num
                	chown $user:$group $file
                	mv /tmp/$file $path
                        cmp -s $path$file $path${file}.check
                        if [ "$?" -eq 1 ]; then
                    cat <<EOF >> /tmp/change.html
                ===========================================================================================
                        <html>
                        <body>
                        <br><b>|$action WARNING|</b> -- File $path$file has modified recently
                        <br><br>Here is the changed lines:
                <pre style="color:red"><code>`diff $path$file $path${file}.check`</code></pre>
                <br>If you are the one doing this action please move ${file}.check to $file
                        </body>
                        </html>
                        ===========================================================================================
EOF
                        fi
                	else
                        	a=$((a-1))
                        	extract $a
        	fi
        else
                echo "Can't find that file in repo"
        fi
}

#If file has php extension modified, move it to check extension
#and restore this file from repo. Also, a warning mail will be sent
if [[ "${file}" =~ .*php$ && "$action" == "MODIFY" ]]; then
	cd $path
    mv $path${file} $path${file}.check
	cd /tmp
	extract
fi

#Find that file in ACL if file doesn't exist in ACL, move that file to check extension
if [[ "${file}" =~ .*php$ && "$action" == "CREATE" ]]; then
	echo "ok"
	grep -x "$path$file" /tmp/acl > /tmp/file_access
    [ -s /tmp/file_access ]
    if [ "$?" -eq 1 ]; then
		mv $path${file} $path${file}.check
	fi
fi

if [[ "${file}" =~ .*php$ && "$action" == "MOVED_TO" ]]; then
    file_php=$file
fi

#If file moved from check extension to php extension, repo will be updated
#A warning mail will be sent
if [[ "${file}" =~ .*check$ && "$action" == "MOVED_FROM" ]]; then
	a=$((a+1))
	cd $docroots
	cd ..
	borg create $repo::"version $a" $path$file_php
	/usr/bin/mailx -S smtp=$smtp_server -S from=$display_name -S smtp-auth=login -S smtp-auth-user="$smtp_username" -S smtp-auth-password="$smtp_pass" -S ssl-verify=ignore -S nss-config-dir=/etc/pki/nssdb/ -s "Wordpress Report" $send_to <<< "|CONFIRM WARNING| -- File '$file' has been confirmed
Server Name: `hostname`
IP: `hostname -I`
Path: $path$file"
fi
done
```
### 6. Run inotify.sh as service
```
cd /etc/systemd/system/
vim noti.service
```
**Copy lines below**
```
[Unit]
Description=My daemon

[Service]
ExecStart=/path/to/script
Restart=on-failure

[Install]
WantedBy=multi-user.target 
```
**Start service**
```
systemctl start noti
systemctl enable noti
```
