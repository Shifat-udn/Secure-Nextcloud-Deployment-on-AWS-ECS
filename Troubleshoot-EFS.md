## Elastic File System Troubleshoot

To see and update files in EFS mount it to a EC2 on the same subnet 

<p>For a Ubuntu EC2, mount the NFS system through NFS client</p>

make directory 

```sh
mkdir ~/efs-mount-point
```
Install NFS Client

```sh
sudo apt update
sudo apt install nfs-common
```
 Mount using client

 ```sh
sudo mount -t nsg4 -o nfsvers=4.1,rsize=1048576,wsize=10488576,hard,timeo=600 xxx.efs.us-east1.amazoneaws.com:/ ~/efs-mount-point

```

now you can run commands like this to get and idea about size 

```sh

du -sh /foldername

 ```

or delete its contect 

```sh

sudo rm -rf directory_name/*

 ```
 
