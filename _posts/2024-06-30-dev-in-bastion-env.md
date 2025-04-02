---
title: Developing in a Restricted Network and Managing Libraries
description: Learn how to install Python libraries and Docker images on a no-internet bastion host using manual transfers securely and easy!
date: 2024-06-30 19:00:00 +0000
future: true
media_subpath: /assets/media/posts/2024-06-30-dev-in-bastion-env/
image: dev-in-bastion-env.jpg
categories: [Posts, Development]
tags: [dev, python, docker]
---

## Or how to get Libraries You Need in there

Some time ago, I faced a challenge while trying to install Python libraries on a bastion host for analyzing vulnerability statistics.
In simple terms, a bastion host is a server with no direct Internet access.
This means, no proxy, no WiFi, no external DNS resolution, and no IP routes outside the LAN.
The only way to transfer files is via SMB, USB flash drive, or other offline methods.

This lack of Internet connectivity makes it difficult to install libraries within an IDE for development, leading to challenges such as:
* No direct access to package repositories (e.g., PyPI, npm, or apt).
* Complicated dependency management when working in isolated environments.
* The need for secure file transfer methods to maintain system integrity.

While this setup may feel inconvenient, it enhances security.

The solution I came up with is simple, yet some developers on Stack Overflow frequently ask how to do it manually. So, here’s my recipe for installing libraries in an offline bastion host environment:

1. Download the required libraries and all dependencies on another machine with Internet access.
2. Scan this files with an antivirus and other security tools to ensure they are safe.
3. Transfer this files to the bastion host using a secure method (USB, SMB, or another safe method).
4. Import the libraries into your IDE and integrate them into your project.

### Example: Downloading Python Libraries for a Bastion Host
Let’s look at an example of downloading the [JupyterLab](https://jupyter.org/) `notebook` package, along with all its dependencies, in `.whl` format for offline installation.

Run the following command on a machine with Internet access:
```bash
pip download notebook --platform win_amd64 --only-binary=:all: -d notebook --python-version 3.13
```
This command will:
* Download the notebook package and all its dependencies.
* Store them in the notebook/ directory.
* Ensure compatibility with Python 3.13 and the Windows AMD64 platform.

Once downloaded, simply transfer the `notebook/` directory to your isolated host and install the packages locally.

Install the package using pip with the `--no-index` flag while inside that directory.
```bash
pip install notebook.whl --find-links ./ --no-index
```
This tells pip not to try to fetch packages from the internet, and the --find-links flag points to your local directory
where the packages are stored.


### Using Local Docker Images for an Offline Bastion Host
Another effective approach is to use Docker to package and transfer pre-configured environments. 
This method allows us to deploy containerized applications on the bastion host without needing direct internet access.

On a machine with an internet connection, pull the required Docker image:
```bash
docker pull quay.io/jupyter/scipy-notebook
```

Run the pulled image and verify its functionality:
```bash
docker run -p 10000:8888 quay.io/jupyter/scipy-notebook
```

To save the container state and create a local image in local repository(127.0.0.1), use:
```bash
docker commit -m commit-$(date +%m-%d-%Y) 6363133417cc 127.0.0.1/scipy-notebook
```
Note: `$(date +%m-%d-%Y)` adds the current date to the commit message for better tracking.

Now, save the image as a compressed archive for easy transfer:
```bash
docker save 127.0.0.1/scipy-notebook | gzip > scipy-notebook_$(date +%m-%d-%Y).tar.gz
```

Move the scipy-notebook_$(date +%m-%d-%Y).tar.gz file to your bastion host using any offline method.

Once the archive is on the bastion host, load the image with:
```bash
docker load -i scipy-notebook_$(date +%m-%d-%Y).tar.gz
```

Verify that the image has been successfully imported:
```bash
docker images
```

### Final Thoughts

Working on a bastion host without internet can be frustrating but with these simple steps, we can keep workflow smooth, even in a locked-down setup.
