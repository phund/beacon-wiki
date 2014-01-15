```
#!/bin/bash
# This Script will Install Some of the software's we need for Ruby Development system.
# We need to add the 02proxy for apt-cache manually 
# Packages like java,spark, google chrome, vim, byobu, htop, pidgin, sambashare setup will be Downloaded from Out Office Samba file Server ftp://192.168.1.15/arrivu_contents/ 
set -x
#Adding IP Information 

echo  "auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
        address 192.168.1.160
        netmask 255.255.255.0
        gateway 192.168.1.1
        network 192.168.1.0
        broadcast 192.168.1.255
        dsn-nameservers 8.8.8.8 192.168.1.1"  > /etc/network/interfaces

#Change the Hostname

echo "ubuntu.example.com" > /etc/hostname
hostname -F /etc/hostname

#After that Change the Host name to system xx to system xx in

echo "# Set the host name here to resolv the Hostname 
127.0.0.1	localhost
192.168.1.160	ubuntu.example.com" > /etc/hosts


#Set the Resolv.conf
 
echo "# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN

nameserver 192.168.1.1
nameserver 8.8.8.8" > /etc/resolv.conf

#set the Resolv.conf Base file 


echo "# Here if we set the Resolv then it wont rewrite the DNS names repeatly after a Reboot

nameserver 192.168.1.1
nameserver 8.8.8.8" > /etc/resolvconf/resolv.conf.d/base


# Restart the Network to take effect before other actions 


sudo /etc/init.d/networking restart


#Update the Repository


sudo apt-get update


# Install the Software Properties Wiht Phython


sudo apt-get install python-software-properties -y


# Install the Advanced vi Editor 


sudo apt-get install vim -y

#Installing Google Chrome

#Install the dependencies for chrome


sudo apt-get install libasound2 libnspr4 libnss3 libxss1 xdg-utils -y

wget ftp://192.168.1.15/arrivu_contents/google-chrome-stable_current_amd64.deb

dpkg -i google-chrome-stable_current_amd64.deb


# Installing Java Manually 


sudo mkdir -p /usr/lib/jvm

wget ftp://192.168.1.15/arrivu_contents/jdk-7u40-linux-x64.tar.gz

sudo mv jdk-7u40-linux-x64.tar.gz /usr/lib/jvm

cd /usr/lib/jvm

sudo tar -zxvf jdk-7u40-linux-x64.tar.gz

sudo rm jdk-7u40-linux-x64.tar.gz

ls -l

sudo update-alternatives --install "/usr/bin/javac" "javac" "/usr/lib/jvm/jdk1.7.0_40/bin/javac" 1

sudo update-alternatives --install "/usr/bin/java" "java" "/usr/lib/jvm/jdk1.7.0_40/bin/java" 1

sudo update-alternatives --set "javac" "/usr/lib/jvm/jdk1.7.0_40/bin/javac"

sudo update-alternatives --set "java" "/usr/lib/jvm/jdk1.7.0_40/bin/java"


# Add the following entries to the bottom of your /etc/profile file:

echo "JAVA_HOME=/usr/lib/jvm/jdk1.7.0_21 PATH=$PATH:$JAVA_HOME/bin export JAVA_HOME export PATH" > /etc/profile

. /etc/profile

java -version

# Navigate to Home Directory 

cd ~

# Install the Byobu Terminal 


sudo apt-get install byobu -y


# Install the htop 


sudo apt-get install htop -y

# Install Pidgin Messenger 

sudo apt-get install pidgin* -y


# Install the Sprak Messenger 

sudo apt-get install ia32-libs -y

wget ftp://192.168.1.15/arrivu_contents/spark_2_6_3.tar.gz

sudo tar -zxvf spark_2_6_3.tar.gz

sudo mkdir /opt/spark

sudo mv Spark/* /opt/spark/


sudo echo "[Desktop Entry]
Name=Spark
Version=2.6.3
GenericName=Spark
X-GNOME-FullName=Spark
Comment=ignite realtime Spark IM client
Type=Application
Categories=Application;Utility;
Path=/opt/spark
Exec=/bin/bash Spark
Terminal=false
StartupNotify=true
Icon=/opt/spark/spark.png
TargetEnvironment=Unity" > /usr/share/applications/spark.desktop


cd /opt/spark

wget https://dl.dropbox.com/u/50880014/spark.png

./Spark &

# Install and Configure Samba Share While every boot 

sudo apt-get install smbfs -y

echo "username=sysadmin
password=admin123$" > /home/sysadmin/.smbcredentials

chmod 0600 /home/sysadmin/.smbcredentials

mkdir -p /home/sysadmin/samba

chown sysadmin:sysadmin /home/sysadmin/samba

echo "//192.168.1.15/public /home/sysadmin/samba cifs credentials=/home/sysadmin/.smbcredentials,uid=sysadmin,gid=sysadmin 0 0" >> /etc/fstab

mount -a

echo "Installtion Completed"

exit
```