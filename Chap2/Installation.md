# INSTALLATION
The Docker installation package available in the official Ubuntu repository may not be the latest version. 
To ensure we get the latest version, we'll install Docker from the official Docker repository. To do that, 
we'll add a new package source, add the GPG key from Docker to ensure the downloads are valid, and then 
install the package

First, update your existing list of packages:
    `$ sudo apt update`

Next, install a few prerequisite packages which let apt use packages over HTTPS:
    `$ sudo apt install apt-transport-https ca-certificates curl software-properties-common`

Then add the GPG key for the official Docker repository to your system:
    `$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

Add the Docker repository to APT sources:
    `$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"`

Next, update the package database with the Docker packages from the newly added repo:
    `$ sudo apt update`

Make sure you are about to install from the Docker repo instead of the default Ubuntu repo:
    `$ apt-cache policy docker-ce`

Notice that docker-ce is not installed, but the candidate for installation is from the Docker repository 
for Ubuntu 18.04 (bionic).

Finally, install Docker:
    `$ sudo apt install docker-ce`

Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it's 
running:
    `$ sudo systemctl status docker`