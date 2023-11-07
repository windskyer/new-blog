---
title: debian-docker-ce
categories: Kubernetes
sage: false
date: 2019-05-10 10:28:09
tags: docker
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# Install docker-ce for Ubuntu/Debian

Uninstall old versions

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```
<!-- more -->

Install Docker CE Pre

>1. Update the apt package index:

```bash
sudo apt-get update
```

>2. Install packages to allow apt to use a repository over HTTPS:

```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common
```

>3. Add Docker’s official GPG key:

```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```

Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint.

```bash
sudo apt-key fingerprint 0EBFCD88

pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
```

>4. Use the following command to set up the stable repository. To add the nightly or test repository, add the word nightly or test (or both) after the word stable in the commands below. Learn about nightly and test channels.

```bash
 sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```

Install Docker CE

>1. Update the apt package index.

```bash
sudo apt-get update
```

>2. Install the latest version of Docker CE and containerd, or go to the next step to install a specific version:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

>3. To install a specific version of Docker CE, list the available versions in the repo, then select and install:

a. List the versions available in your repo:

```bash
apt-cache madison docker-ce

  docker-ce | 5:18.09.1~3-0~debian-stretch | https://download.docker.com/linux/debian stretch/stable amd64 Packages
  docker-ce | 5:18.09.0~3-0~debian-stretch | https://download.docker.com/linux/debian stretch/stable amd64 Packages
  docker-ce | 18.06.1~ce~3-0~debian        | https://download.docker.com/linux/debian stretch/stable amd64 Packages
  docker-ce | 18.06.0~ce~3-0~debian        | https://download.docker.com/linux/debian stretch/stable amd64 Packages
  ...
```

b. Install a specific version using the version string from the second column, for example, 5:18.09.1~3-0~debian-stretch .

```bash
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```

>4. Verify that Docker CE is installed correctly by running the hello-world image.

```bash
sudo docker run hello-world
```