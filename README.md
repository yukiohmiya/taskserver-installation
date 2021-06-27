# Taskserver Installation for Ubuntu 20.10
## Preparation
### Add User, Group
```sh
sudo su
adduser taskd
```
### Update
```sh
apt update;apt upgrade
```
### Install Dependencies
```sh
apt install build-essential cmake uuid-dev gnutls-dev python-is-python3 gnutls-bin
```

## Installation
### Clone Repository
```sh
cd ~
git clone https://github.com/GothenburgBitFactory/taskserver.git
mv taskserver /usr/local/taskd
chown -R taskd:taskd /usr/local/taskd
cd /usr/local/taskd
git chekout 1.2.0
git submodule init
git submodule update
```
### Build
```sh
cd /usr/local/taskd
cmake -DCMAKE_BUILD_TYPE=release .
make
```
### Test
```sh
cd test
make
./run_all
cd ..
```
### Install
```sh
make install
```
### Verify
```sh
taskd

Usage: taskd -v|--version
       taskd -h|--help
       taskd diagnostics
       taskd validate <JSON | file>
       taskd help [<command>]

Commands run only on server:
       taskd add     [options] org   <org>
       taskd add     [options] group <org> <group>
       taskd add     [options] user  <org> <user>
       taskd config  [options] [--force] [<name> [<value>]]
       taskd init    [options]
       taskd remove  [options] org   <org>
       taskd remove  [options] group <org> <group>
       taskd remove  [options] user  <org> <user>
       taskd resume  [options] org   <org>
       taskd resume  [options] group <org> <group>
       taskd resume  [options] user  <org> <user>
       taskd server  [options] [--daemon]
       taskd status  [options]
       taskd suspend [options] org   <org>
       taskd suspend [options] group <org> <group>
       taskd suspend [options] user  <org> <user>

Commands run remotely:
       taskd client  [options] <host:port> <file> [<file> ...]

Common Options:
  --quiet        Turns off verbose output
  --debug        Generates debugging diagnostics
  --data <root>  Data directory, otherwise $TASKDDATA
  --NAME=VALUE   Temporary configuration override
```

## Server Setup
### Server Configuration
#### Choose a Data Location
A location for the data must be chosen and created. The TASKDDATA environment variable will be used to indicate that location to all the taskd commands.
```sh
export TASKDDATA=/var/taskd
mkdir -p $TASKDDATA
chown -R taskd:taskd $TASKDDATA
```
#### Initialization
```sh
su taskd
taskd init
```
#### Keys & Certificates
```sh
cd /usr/local/taskd/pki
vim vars
...
CN=localhost	# You have to change domain or localhost not IP address
...
```
I can't run server when I tried to configure Common Name using server IP Address,
You SHOUD change localhost or domain name.

```sh
./generate
...
(generated some key pairs.)
cp *.pem $TASKDDATA

taskd config --force server.cert $TASKDDATA/server.cert.pem
taskd config --force server.key $TASKDDATA/server.key.pem
taskd config --force server.crl $TASKDDATA/server.crl.pem
taskd config --force ca.cert $TASKDDATA/ca.cert.pem
```

#### Configuration
```sh
touch $TASKDDATA/taskd.log
taskd config --force log $TASKDDATA/taskd.log

taskd config --force pid.file /var/taskd.pid
taskd config --force server localhost:53589
```

### Server Start/Stop
```sh
taskd server
...
Ctrl+C

```

### Configure Taskserver to run with a systemd-unit-file
```sh
cd ~
vim taskd.service
```

```sh
[Unit]
Description=Secure server providing multi-user, multi-client access to Taskwarrior data
Requires=network.target
After=network.target
Documentation=https://taskwarrior.org/docs/#taskd

[Service]
ExecStart=/usr/local/bin/taskd server --data /var/taskd
Type=simple
User=taskd
Group=taskd
WorkingDirectory=/var/taskd
PrivateTmp=true
InaccessibleDirectories=/home /root /boot /opt /mnt /media
ReadOnlyDirectories=/etc /usr

[Install]
WantedBy=multi-user.target
```

```sh
cp taskd.service /etc/systemd/system
systemctl daemon-reload
systemctl start taskd.service
systemctl status taskd.service
```

```sh
systemctl enable taskd.service
```

## Client Setup
### Create Organization
```sh
taskd add org Public
```

### Create User
```sh
taskd add user Public first_last

New user key: 755ea65f-e1c4-444e-a4bc-a9be4e96a7f0
Created user 'first_last' for organization 'Public'

```
**New user key** is important value for user identification.

```ssh
cd /usr/local/taskd/pki
./generate.client first_last

cp first_last.key.pem first_last.cert.pem $TASKDDATA
```

## Sync
### Configure Taskwarrior
```sh
cp first_last.cert.pem ~/.task
cp first_last.key.pem  ~/.task
cp ca.cert.pem         ~/.task
```

```sh
task config taskd.certificate -- ~/.task/first_last.cert.pem
task config taskd.key         -- ~/.task/first_last.key.pem
task config taskd.ca          -- ~/.task/ca.cert.pem
task config taskd.server      -- localhost:53589
task config taskd.credentials -- Public/first_last/755ea65f-e1c4-444e-a4bc-a9be4e96a7f0
```

```sh
task sync init
```
