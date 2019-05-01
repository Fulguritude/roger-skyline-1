Roadmap for roger-skyline-1 for Debian (various tests with Debian Server, Xubuntu and Lubuntu), with inspiration from DMaxence's, jmoussu's, and nvienot's original guides.

Remember to take frequent snapshots of your VM, and don't be afraid to start over from scratch if need be. Note that for many distros, the `service restart` type commands can return OK and yet fail, so it's best practice to fully reboot at each config change in order to not confuse oneself thinking something has changed when it has not.

Man pages and software to download:
- sudo
- su
- adduser
- usermod
- interfaces
- ifconfig
- ip (a, route, ..)
- ifup/ifdown/ifquery
- iptables
- dhclient
- portsentry
- fail2ban
- nmap (`brew install nmap` on MacOS host)

Files:
- /etc/netplan/\*.yaml
- /etc/network/interfaces
- /etc/init.d/cron (or `sudo crontab -e`)
- /etc/ssh/sshd_config
- /etc/fail2ban/jail.conf
- /etc/fail2ban/jail.local
- /etc/default/portsentry
- /etc/portsentry/filter.d/\*.conf
- /etc/hosts.deny
- /var/log/syslog
- /var/log/\*

Links:
https://medium.com/platform-engineer/port-forwarding-for-ssh-http-on-virtualbox-459277a888be

https://netplan.io/examples

https://websiteforstudents.com/configure-static-ip-addresses-on-ubuntu-18-04-beta/

https://blog.teamtreehouse.com/set-up-a-linux-server-on-virtualbox

https://www.linux.com/learn/intro-to-linux/2018/9/how-use-netplan-network-configuration-tool-linux

https://doc.fedora-fr.org/wiki/SSH_:_Authentification_par_cl%C3%A9

https://www.aidoweb.com/tutoriaux/changer-port-serveur-ssh-645


## #1 : **Getting OS packages up-to-date**

If sudo isn't already installed by default (like in Debian server), run:

`su root` or `su -`
`apt install sudo`
`exit`

Note that if your version of Debian is old, apt-get rather than apt might be required.
To put packages up to date, run:

`sudo apt update`

This queries the software repository database to see if current OS build is up to date. It creates a list of packages to be upgraded. Older version of `apt` is `apt-get`.

`sudo apt upgrade`

Downloads patches for the different software packages in the list returned by the previous command.


## #2 : **Creating a user**

`sudo adduser [userlogin]`
`sudo usermod -aG sudo [userlogin]`

This adds the user to the "sudoers" file and grants them the administrative rights to use the "sudo" command. "-aG" is a special combined option to append a user to a group.

Alternatively, you can run:

`sudo adduser [userlogin] sudo`

And this should have the same result as the second line, adding the user to the system "sudo" group.

Note the distinction between sudo, which grants admin rights for a single command, and "su -/-l/--login" without an argument which turns every following command into a "root" command, ie, full admin rights without having to write "sudo" before every command.

Finish with:

`su - [userlogin]`

and enter the password for [userlogin] to log in as the newly created user.


## #3 : **Configuring the network interfaces**

On most Debian, you'll find what you need in `/etc/network/interfaces` or in more recent builds, `/etc/netplan/...`. To configure the network, we can edit /etc/network/interfaces and execute `/etc/init.d/networking restart` (or edit `/etc/netplan/....yaml` and execute `netplan apply`).

If you change the line `iface eth0 inet dhcp` with `iface eth0 inet static`, it should disable the DHCP protocol (which sets up the interface information automatically).

`ip link show` or `ip a` will give you information concerning available interfaces (especially if ifconfig is unavailable).

Then add the following lines to the `/etc/network/interfaces` file for the corresponding interface on the VM:

```
auto enp0s3
iface enp0s3 inet static
    address 10.1[floor_digit].[nb_greaterthan_nb_of_cluster_rows].[nb_greaterthan_nb_of_cluster_columns]
    netmask 255.255.255.252
    broadcast 10.1[floor_digit].255.255
    gateway 10.1[floor_digit].254.254
```

where "enp0s3" should be the appropriate interface (the second for the "ip a" command, or rather, the one with BROADCAST) on your VM. These lines create a home address for enp0s3, and divides the newly made host into a network with 4 sub IDs (your 252 is 0xFC in hex, meaning the netmask leaves only 2 free bits, ie, a netmask of /30, over an IPv4 address which is 32 bits).

Restart networking with `/etc/init.d/networking restart` or `netplan apply` if you're feeling lucky; or else just reboot the guest OS/VM with `sudo reboot`.


## #4 **Changing the SSH port**

Open the file `/etc/ssh/sshd_config`.

Modify `#Port 22` to the desired port 'MY_PORT', which should be a number above 1024 and below 65535. Uncomment the line.

Change `#PermitRootLogin [...]` to `PermitRootLogin no`

Change `#PasswordAuthentication [...]` to `PasswordAuthentication no` (if you want to use `scp` rather than `ssh-copy-id`, you might need to keep `yes` here until you've copied the key).

Restart ssh with `sudo /etc/init.d/ssh restart`, or `sudo reboot` if you want to be safe because you don't trust your distro.

Generate a **public key** on the host with `ssh-keygen` command.

Copy the file `~/.ssh/id_rsa.pub` from the host to the location `~/.ssh/authorized_keys` in the VM user's home directory (`scp` is a bit of work but useful to know; or `ssh-copy-id -i ~/.ssh/id_rsa.pub [user]@[vm-IP] -p [MY_PORT]` which is)


## #5 **Firewall**

Everything revolves around iptables. ufw is just a bunch of scripts to handle iptables better. See DMaxence for a good use of iptables. Or else, just look up ufw.

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 443
sudo ufw allow 80/tcp
sudo ufw allow 25
sudo ufw allow [MY_PORT]
```

Note that port 25 is SMTP, 80/tcp is HTTP (only for TCP not UDP), 443 is HTTPS, and MY_PORT is SSH. 


## #6 **DoS protection**

ufw & fail2ban; test with pingflood, synflood, slowloris


## #7 **Port scan protection**

portsentry, test from host with `nmap [vm-IP]` and `nmap -Pn [vm-IP]`. After each test (they are long), check the file `/etc/hosts.deny` for the host IP (it should have been added by portsentry), remove it, and reboot, or else you won't be able to ssh into the vm from the host.


## #8 **Stopping useless services**

`sudo service --status-all`, get acquainted with all of them, especially the ones that you can't stop without breaking the machine. Normally, with a Debian Server install, you've got pretty much nothing to disable here.


## #9 **Automatically updating packages**

Create a script:

`sudo vim /root/scripts/update_script.sh`

With the following lines:

```bash
#!/bin/bash
apt update -y >> /var/log/update_script.log
apt upgrade -y >> /var/log/update_script.log
```
Give it some permissions:

`sudo chmod 755 /root/scripts/update_script.sh`

And make root be the owner for automated execution:

`sudo chown root /root/scripts/update_script.sh`

To automate execution, we must edit the crontab file:

`sudo crontab -e`

To which we add the lines:

```
0 4 * * wed root /root/scripts/update_script.sh
@reboot root /root/scripts/update_script.sh
```
The crontab file explains its format in a large comment. The format is straightforward.


## #10 **Making a script to warn of all crontab edits**

Create a script :

`sudo vim /root/scripts/crontab_canary.sh`

With the following lines :

```bash
#!/bin/bash

DIFF=$(diff /etc/crontab.bak /etc/crontab)

cat /etc/crontab > /etc/crontab.bak

if [ "$DIFF" != ""] then
    echo "crontab check: changed, notifying admin."
    sendmail root@127.0.0.1 < ~/mail.txt
else
    echo "crontab check: unchanged."
fi
```

Give it the appropriate execution permessions:

`sudo chmod 755 /root/scripts/script_crontab.sh`

Make it owned by root :

`sudo chown root /root/scripts/script_crontab.sh`

And edit crontab by adding the following line:

```
0 0 * * * root /root/scripts/crontab_canary.sh
```

Don't forget to make your `~/mail.txt` file !
