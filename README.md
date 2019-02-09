# Installing a Raspberry Pi Home Server

This tutorial is a living document as I experiment with adding new software to run on my Raspberry Pi. I have found that using the Rasberry Pi in this way can be useful for quick prototyping purposes (e.g. Flask apps, PostGIS queries, etc.).

At a minimum, this tutorial covers how to:
1. Mount an external harddrive and make it network accessible to Windows or Linux machines
2. Set up a personal, self-hosted, and offline Git repo using Gogs. (Note: there's not much point to do this these days now that GitHub allows free private repos)
3. Create automated backups of files on your Pi or other storage accessible by the Pi (i.e. externally mounted harddrives)

All commands are executed from the Raspberry Pi terminal unless otherwise stated.

## Initial setup
---
1. Start with installing all apts that will be used:

        sudo apt-get install rsync postgresql samba gparted ntfs-3g postgresql-client libpq-dev
    * At this point, use gparted to do any partitioning you may want to do on the external harddrive. For example, I have partitioned 64GB in ext4 format of my 250GB harddrive as a back-up for the files on my SD card. The remaining space I formatted in ntfs and will mount to access over the network. According to the tutorial I used to set up automated back-ups using rsync, you will want to have a backup footprint of approximately twice the storage capacity you are backing up.

## Externally mounted storage
---
1. Set up any external harddrive mounts that will be used for shared storage
    * Find the location of the harddrive to mount using:
   
          sudo fdisk -l
    * Make the directory for mounting, mine will go in /mnt/PIHDD1
    
          sudo mkdir /mnt/PIHDD1
    * Mount the harddrive (make sure it's plugged in). If you have multiple external storage devices, you may have to use *sudo fdisk -l* to figure out the correct path and change */dev/sda1* accordingly.

           mount /dev/sda1 /mnt/PIHDD1
2. Change folder permissions and ownership


        sudo chmod 0777 /mnt/PIHDD1
        sudo chown -R pi /mnt/PIHDD1
3. Edit the Samba config file, which defaults into */etc/samba/smb.conf*

       sudo nano /etc/samba/smb.conf 
    
    Add in the following to the end of the config file:

        [PIHDD1]

        Comment = Shared
        Path = /mnt/PIHDD1
        Writeable = Yes
        only guest = no
        create mask = 0777
        directory mask = 0777
        Public = yes
        Guest ok = yes
        force user = pi

4. Restart the samba server, using 
     
       sudo service smbd restart    
     The directory and should be readable and writeable by anyone on the network
5. Edit fstab in order for the external to mount automatically on boot up. You will need the UUID for the device for this, which you can find using fdisk -l and blkid (both run as sudo), you should be able to find the UUID for the external harddrive partition you want to mount.

       UUID=*YOURDEVICESUUID* /mnt/PIHDD1 ntfs defaults,auto,umask=000,noatime,nofail,users,rw 0 0

## Set up a personal git repository using gogs
---
This portion of the tutorial closely follows: https://dev.to/\_lunacy\_/private-github-with-gogs-and-raspberry-pi-46m3. The tutorial differs from the linked in that it uses Postgresql as the underlying database for the gogs deployment, since I plan to use Postgres for its PostGIS extension. There are a few changes that need to be made from the tutorial is you also intend to use Postgres that I mention below. Now that GitHub allows for unlimited private repositories, this part of the tutorial may not be particularly useful.

1. Have Postgresql server start on boot-up. The third line is where you might try to change the location of the database initialization. By default it is /var/db/postgres

        sudo systemctl start postgresql
        sudo systemctl enable postgresql
        service postgresql initdb
2. Set up the database that the gogs installer will use

        sudo -u postgres psql -d template1
    This will open up the psql terminal in your current window, then run line-by-line:

        CREATE user gogs CREATEDB;
        \password gogs
        ... enter a password...
        CREATE DATABASE gogs OWNER gogs;
        \q

3. Download gog installer. You may need to update the link in the wget statement to the newest LOCAL: ZIP for the Raspberry Pi at: https://gogs.io/docs/installation/install_from_binary

        wget https://dl.gogs.io/0.11.66/gogs_0.11.66_raspi2_armv6.zip
        unzip gogs_0.11.66_raspi2_armv6.zip
        cd gogs
        ./gogs web
4. From here, go to *0.0.0.0:3000* using a web browswer on the pi. Change the user and password in the first chunk of options to gogs and the associated password. In the next chunk of credentials, make sure that the run user is set to pi (or your current user in the terminal). You also probably want to check use built-in ssh.


5. Run the following in order to be able to access the webpage from anothere computer on your network. Make sure to change /pi/gogs/gogs to the actual path of the **executable**:

        sudo setcap CAP_NET_BIND_SERVICE=+eip home/pi/gogs/gogs
6. Edit the SSH port so there are no conflicts with gogs which uses PORT 22.

        sudo nano /etc/ssh/sshd_config
    Remove the '#' in the line: #Port 22 and make it something else. Restart the ssh service.

        sudo service ssh restart
7. Test out that the connection works. First, get the internal ip of the raspberry pi by running *ifconfig* in the terminal. 
![ifconfig output](/images/internal_ip.JPG)
8.  Then, from the gogs folder run 

        ./gogs web
    If all has gone well, you should now be able to use another computer on your network to open the gogs web interface by using a browser to go to: *yourpi_ip:3000*.
![Gog initial screen](/images/gogs_splash_initial.PNG)
    You will now need to register. After registering, you can sign and and should see a new repository screen!
![Gog registration screen](/images/gogs_register.PNG)

9.  Finally, we will set up gogs to run as a daemon, so that it will immediately start when the pi is booted up. 

    * First, copy over the handy script that is included with gogs.

            sudo cp /home/pi/gogs/scripts/init/debian/gogs /etc/init.d/gogs
    * Then, edit it:

            sudo nano /etc/init.d/gogs
        Specifically, we will want to change the working directory to match where gogs was installed and the USER, circled below. **This is not the database user, but the used who owns the gogs installation. I changed both circled areas to *pi* in my case.**
![Daemon setup](/images/gogs_script_changes.png)

    * Make sure you closed the terminal that was previously running ./gogs web, then run the following commands in order to finalize the changes we just made. The link at the beginning of this section has good descriptions of why we do these steps

            sudo chmod ug+x /etc/init.d/gogs
            sudo update-rc.d gogs defaults 98
            sudo service gogs start
        If everything went according to plan, you should now be able to access the web interface from any other computer on your network. The gogs server will always be ready and waiting when your pi is booted up. 

## Automated back-ups using rsync
---
This part of the tutorial follows: http://www.mikerubel.org/computers/rsync_snapshots/#Rsync. I have changed the frequency and location of the automated back-ups, and included a few crucial steps. If your set-up is similar to mine, you will want to follow the changes I made closely.

1. Move the excludesync and includesync files into /etc/. Alter them as you see fit, the current sync affects almost everything on the SD card. If you are following along, it is crucial you have a similar exclude file. I got stuck in an infinite syncing loop when I forget to exclude /mnt/* or /media/*.

2. Change the parameters in daily_snapshot_rotate.sh and make_snapshot.sh to match the folder paths on your system. Put these files somewhere that the cron job can run them. I put these files into a folder I created, /etc/rsync/.

3. Make relevant changes to the make_snapshot and rotate_daily_snapshot shell scripts found near the end of the link below. Specifically, you will watn to change *MOUNT_DEVICE*, *SNAPSHOT_RW* paths in both files. You will also need to make sure the paths to *INCLUDES* and *EXCLUDES* match where you put those files in the two files.

4. If you are setting all of this up on a Pi and running as the default user, there is one detail required to get the setup to work properly. Run the following code to change the permission of the .sh scripts, then give it a whirl by running the next code block while in the directory with the shell script:

        sudo chmod +x /etc/rsync/make_snapshot.sh
        sudo chmod +x /etc/rsync/rotate_daily_snapshot.sh
        sudo ./make_snapshot.sh

    It might take a few minutes for this part to run. The console might hang up for a moment before it even starts. You should soon see a spike in processor activity for the next few minutes.

5. Change /etc/crontab with nano by adding rows that specify when to run the shell scripts. I added the following for my purposes:

        0 21 * * * root /etc/rsync/make_snapshot.sh
        0 10 * * 0 root /etc/rsync/daily_snapshot_rotate.sh
        0 22 * * 0,3 root /etc/rsync/daily_snapshot_rotate.sh

    These settings make a snapshot every night at 9PM, and do a longer-term rotation snapshot every Wednesday at 10PM and Sunday at 9AM.
