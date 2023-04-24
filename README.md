
**Summary**

This explains how to enable a docker container and the host both be able to write the same mounted directory.

1. In the container Dockerfile enable group write permissions flags with `umask 002`.
2. On the host side:
    1. Create a new group with a special id, and add the host user that group.  
    2. Set the directory and file permissions of the targeted directory and its subdirectories. 

**Dockerfile**

```
FROM ubuntu
ARG USERNAME=username
ARG USER_UID=1000
ARG USER_GID=$USER_UID
# Add user
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME
# Enable group write flags for every file the user creates
RUN su $USERNAME -c "echo 'umask 0002' >> /home/${USERNAME}/.bashrc"
```

**Host side: new group**


Note:  In the testing environment, `ubuntu 20.4`, used by this author, the group id of files created by username which was `1000` inside the container, appears as `100999` outside of the container.  This is likely to vary by system.

On the host create a group with gid 100999, and add the host user to that group.  It is necessary to log out and log back in again for it to take effect. (*)

```
sudo addgroup --gid 100999 g100999
sudo usermod -a -G g100999 craig
```

**Host side: target directory and file settings**

Ensure the whole directory and all its subdirectories have group read, write, executable, and setuid permissions.  (They probably already do for rwe, but maybe not s).
```
sudo find . -type d -exec chmod g+rwxs {} +
```

Ensure the all files in all subdirectories have group read and write permissions:
```
sudo find . -type f -exec chmod g+rw {} +
```

**Create the container**

```
docker build -t testi .
docker run --rm -u 1000:1000 -d -it --name testc --mount type=bind,source="$(pwd)",target=/targetdir testi
docker attach testc
```

From within the container

```
username@154d42ef5260:/$ cd targetdir
username@154d42ef5260:/targetdir$ ls -alt
total 24
drwxr-xr-x 1 root     root     4096 Apr 24 16:34 ..
drwxrwsr-x 2 username username 4096 Apr 24 16:12 .
-rw-rw-r-- 1 username username  432 Apr 24 15:56 Dockerfile
```

From the host outside the container the same directory looks like

```
$ ls -alt
drwxrwsr-x 91 craig  craig   4096 Apr 22 15:41 ..
drwxrwsr-x  2 100999 g100999 4096 Apr 24 09:12 .
-rw-rw-r--  1 100999 g100999  432 Apr 24 08:56 Dockerfile
```

You should be able to create, write, and delete the files in that directory from either the host or the container sides. 


