# raspberry-pi-restic-backup
Scripts and setup to backup rasberry pi using restic to nfs


## Setup

Install restic

```bash
wget https://github.com/restic/restic/releases/download/v0.12.0/restic_0.12.0_linux_arm.bz2
bzip2 -d restic_0.12.0_linux_arm.bz2
chmod +x restic_0.12.0_linux_arm
mv restic_0.12.0_linux_arm /usr/local/bin
ln -s /usr/local/bin/restic_0.12.0_linux_arm restic
which restic
```

## Init repo

Make sure you nfs share is mounted. Here I have mine mounted to /mnt/storage1

export RESTIC_REPOSITORY=/mnt/storage1/$(hostname)
export RESTIC_PASSWORD_FILE=/home/pi/.restic/restic-pw.txt

mkdir -p /home/pi/.restic/
echo "your_secure_password" > /home/pi/.restic/restic-pw.txt
chmod 600 /home/pi/.restic/restic-pw.txt

## Config include/excludes


```bash
cat > /home/pi/.restic/include.txt << 'END'
/boot  
/etc  
/home  
/lib  
/media  
/opt   
/root  
/sbin  
/srv  
/sys  
/usr  
/var
END
```

```bash
cat > /home/pi/.restic/exclude.txt << 'END'
/tmp
/var/tmp
/var/run
END
```

## Backup Script

```bash
cat > /usr/local/bin/restic_backup.sh << 'END'
#!/bin/bash

export RESTIC_REPOSITORY=/mnt/storage1/$(hostname)
export RESTIC_PASSWORD_FILE=/home/pi/.restic/restic-pw.txt
export INCLUDE=/home/pi/.restic/include.txt
export EXCLUDE=/home/pi/.restic/exclude.txt

function log()
{
    echo "$1" | logger -t restic_backup
}


log "Starting restic backup..."

/usr/local/bin/restic backup --exclude-file $EXCLUDE --files-from $INCLUDE --host $(hostname) | logger -t restic_backup

log "Backup completed"
END
chmod +x /usr/local/bin/restic_backup.sh
```

## Systemd service

```bash
cat > /etc/systemd/system/restic-backup.service << 'END'
[Unit]
Description=Runs restic backup script

[Service]
Type=oneshot
ExecStart=/usr/local/bin/restic_backup.sh
User=root
END
```

## Systemd timer

```bash
cat > /lib/systemd/system/restic-backup.timer << 'END'
[Unit]
Description=Run restic-weekly.service every Monday at 5:30AM

[Timer]
Unit=restic-backup.service 
OnCalendar=Mon *-*-* 05:30:00

[Install]
WantedBy=timers.target
END
systemctl enable restic-backup.timer
```


## List backups

```bash
export RESTIC_REPOSITORY=/mnt/storage1/$(hostname)
export RESTIC_PASSWORD_FILE=/home/pi/.restic/restic-pw.txt

#Find zip files
restic find .zip
```