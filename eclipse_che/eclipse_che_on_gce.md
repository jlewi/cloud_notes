# Instructions for installing Eclipse Che on Google Compute Engine

Some instructions for setting up Eclipse Che on a Google Compute Engine.
Instructions explain how to connect to Eclipse Che over ssh using a socks 
proxy.

## Create a VM

Use GCP's cloud console or the command line to create a VM. 

Some suggested VM settings
* 4 vCPUs
* Ubuntu 15.10 image
* Scopes - full access to Cloud APIs
* 1 TB boot disk
* **disable** delete boot disk on instance delete
* No additional disks
* Allow IP Forwarding

TODO(jlewi) Would it be better to store data on separate PD?

Why ubuntu 15.10?
  * Ubuntu 16.04 didn't work: ran into this issue: 
    https://github.com/eclipse/che/issues/745
  * Ubuntu 14.04 doesn't have JDK8 in the apt repo
  
## Setting up the vm

Follow the instructions on Docker's website for installing Docker
https://docs.docker.com/engine/installation/linux/ubuntulinux/
  * You need to add Docker's apt resource and pull in the docker-engine package (docker in the apt repo is too old for ubuntu 15.10)

```
sudo apt-get update
sudo usermod --append --groups docker "$USER"
sudo apt-get install -y openjdk-8-jdk-headless emacs screen
```
* You will need to relogin for usermod to take effect

* Download Eclipse Che
* Goto
http://www.eclipse.org/che/download/
Click on one of the links to start downloading
	* Make sure you download the latest version (the signed binaries link might point to an older version.)
* If you use chrome and go to the downloads menu you can get the actual file link you can then do
wget <file href>
to download it onto the vm

* Unpack it to

```
/usr/local/eclipse-che-<Version>
```

TODO(jlewi) should we store the workspaces directory outside of eclipse-che-<version> to make it easier to upgrade?

# Setup the UID for eclipse

According to the [instructions](https://eclipse-che.readme.io/docs/usage)
eclipse che wants to run as UID 1000.

Looking at /etc/passwd it looks like GCE had already assigned that to some other user (probably so it could install the ssh keys for that user).

* Edit /etc/passwd and change the user associated with uid 1000 to 'eclipse' and change the home directory to /home/eclipse

* Edit /etc/group and change the corresponding group for that user to 'eclipse

```
sudo addgroup eclipse
sudo usermod --append --groups eclipse "eclipse"
```

* Add the eclipse user to the docker group

```
sudo usermod --append --groups docker "eclipse"
sudo mkdir /home/eclipse
sudo chown -R eclipse /home/eclipse
sudo chgrp -R eclipse /home/eclipse
sudo chown -R eclipse /usr/local/eclipse-che-4.3.2
sudo chgrp -R eclipse /usr/local/eclipse-che-4.3.2
echo 'export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/' >> /etc/bash.bashrc
```

Start eclipse
```
screen
sudo bash
su eclipse
export CHE_DOCKER_MACHINE_HOST=`ifconfig eth0 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}'`

/usr/local/eclipse-che-4.3.2/bin/che.sh run
```


## Connecting From a Remote Host

You need to setup [dynamic port forwarding](https://help.ubuntu.com/community/SSH/OpenSSH/PortForwarding#Dynamic_Port_Forwarding)

Then

```
gcloud compute ssh --project=${PROJECT} --ssh-flag="-C -D 1080" --zone=us-central1-f ${VM_NAME}
```
Then configure you're web browser to use socks.

 * In chrome you can use the exention [chrome-proxy-helper](https://github.com/henices/Chrome-proxy-helper)
	 to configure proxy settings
 * In chrome you can enable the proxy for a different profile

Access it at
```
http://<CHE_DOCKer_MACHINE_HOST>:8080
```

## Custom Recipes And GCR

* Using a private image in GCR, I've had problems if I started with an existing statck and then tried changing to a custom stack using my private image.
* Creating a new workspace and specifying a custom a recipe at workspace creation had no such problem.
* Could be an issue with private GCR or something else.

## Troubleshooting
Logs will be in

/usr/local/eclipse-che-4.0.1/tomcat/logs/

Machine logs are in

```
/usr/local/eclipse-che-4.3.2/tomcat/logs/machine
```

## References

[issues/438](https://github.com/eclipse/che/issues/1438) Question about hosting Eclipse Che on AWS EC2.
