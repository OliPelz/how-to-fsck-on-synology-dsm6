How to properly do a filesystem check on Synology DSM 6.0 e.g. DS414

I tried a lot of instructions and tutorials to do a file system check on a Synology DSM 6 device e.g the DS414.
The first step involves unmounting the partition you want to check e.g. the /volumes/ path before you can file system check it.
All the instructions I found are inaccurate, too old (most are for DSM 4 or 5), do not work or a dangerous. I just could not get the unmounting to work!

Presteps are install ipckg using instructions found here: https://github.com/basmussen/ds414-boostrap-dsm5
then install the packages less, lsof, mlocate, 

E.g. the common advice:
```
syno_poweroff_task -d 
```
shuts down all services including telnet and apache, etc. also shutsdown my ssh server and the webserver making the box completely inaccessible while still powered on -> you need to hard reset the box


the other common advice to just do a 
```
lsof /volumes/
```
and then kill the PID of the processes using the volume. Problem with this is that most services are watched by the system so if you kill them, they just restart again after a sec.

Here is my solution:
Get the list of services associated with your volume you want to fs check:
```
lsof /volumes
```

Or make the list more clear with:
```
lsof /volume1/ | sed 1d | cut -d" " -f1 | sort | uniq
```
e.g.

```
anvil
ash
cnid_dbd
cut
dovecot
img_backu
log
master
php56-fpm
pickup
postgres
qmgr
s2s_daemo
sed
sh
sort
syno_mail
afpd
cnid

```
If you are into a bit of Linux you can spot/group these services into categories:

php5/httpd/apache2/nginx = searchterms httpd,nginx
postgres = searchterms postgres
dovecot/syno_mail = searchterm mail
...


to generally find services by name use the following syntax
```
find /usr/syno/etc.defaults/rc.sysv/ | grep -i <service name>
synoservicecfg --status | grep enable | grep -i <service name>
```
e.g.

```
find /usr/syno/etc.defaults/rc.sysv/ | grep -i postgres
synoservicecfg --status | grep enable | grep -i nginx
```
So my approach was spot a service which sounds promising, stop it and then run 
```lsof /volume1/ | sed 1d | cut -d" " -f1 | sort | uniq``` to see if this service vanishes from the list.
So all in all I found the following services which I had to stop.


#shutdown postgres - postgesql
/usr/syno/etc.defaults/rc.sysv/pgsql.sh stop

#stop php5
synoservicecfg --stop pkgctl-PHP5.6 

#shutdown Mailserver
synoservicecfg --stop pkgctl-MailServer

# shutdown backups  (img_backu)
synoservicecfg --stop synobackupd
synoservicecfg --stop pkgctl-HyperBackupVault
synoservicecfg --stop pkgctl-synobackupd
synoservicecfg --stop pkgctl-HyperBackup
synoservicecfg --stop pkgctl-HyperBackupVault
synoservicecfg --stop pkgctl-TimeBackup

# shutdown s2sdaemon
synoservicecfg --stop s2s_daemon

# others: afp and cnid_dbd 
Since I could not find any service definition file for that I simply killed the processes using good old ```kill``` command, which did not restart luckily within a minute or so.

# now the last thing what was still missing were some user cwd etc. processes connected, as the /home folder was part of the /volumes1 folder:
```
sh      8480  Oli  cwd    DIR  253,1     4096 154796037 /volume1/homes/Oli
sudo    9104 root  cwd    DIR  253,1     4096 154796037 /volume1/homes/Oli
ash     9105 root  cwd    DIR  253,1     4096 154796037 /volume1/homes/Oli
lsof    9209 root  cwd    DIR  253,1     4096 154796037 /volume1/homes/Oli
lsof    9209 root  txt    REG  253,1   125544 369233175 /opt/sbin/lsof
lsof    9210 root  cwd    DIR  253,1     4096 154796037 /volume1/homes/Oli
lsof    9210 root  txt    REG  253,1   125544 369233175 /opt/sbin/lsof
```

Solution here was to logout your user and login the true root user then you can umount
```
umount /opt
umount /volume1
```

#then finally run your fsck diagnostic etc.
```
fsck.ext4 -fv /dev/mapper/vol1-origin
```

done!
