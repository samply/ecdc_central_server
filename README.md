# ECDC Locator

The Locator provides a GUI and a backend for running antibiotic-resistance-related queries against national nodes that have installed the [Bridgehead](https://github.com/samply/bridgehead/tree/ehds2).

The GUI is called Lens, the backend is called Spot.

## Requirements

The data protection group at your site will probably want to know exactly what our software does with patient data, and you may need to get their approval before you are allowed to install these components.

### Hardware

Hardware requirements strongly depend on the specific use-cases of your network as well as on the data it is going to serve. Most use-cases are well-served with the following configuration:

- 2 CPU cores
- 8 GB RAM
- 100GB Hard Drive, SSD recommended

### Software

You are strongly recommended to install the components under a Linux operating system (but see the section [Non-Linux OS](#non-linux-os)). You will need root (administrator) priveleges on this machine in order to perform the deployment. We recommend the newest Ubuntu LTS server release.

Ensure the following software (or newer) is installed:

- git >= 2.0
- docker >= 20.10.1
- docker-compose >= 2.xx (`docker-compose` and `docker compose` are both supported).

We recommend to install Docker(-compose) from its official sources as described on the [Docker website](https://docs.docker.com).

> 📝 Note for Ubuntu: Snap versions of Docker are not supported.

### Network

You will need to establish URLs for Lens and Spot (in the same domain, to avoid CORS problems). This may entail registering the URLs and setting up DNS.

The following URLs need to be accessible on the machine hosting the Locator (prefix with `https://`):
* To fetch code and configuration from git repositories
  * github.com
  * git.verbis.dkfz.de
* To fetch docker images
  * docker.verbis.dkfz.de
  * Official Docker, Inc. URLs (subject to change, see [official list](https://docs.docker.com/desktop/all))
    * hub.docker.com
    * registry-1.docker.io
    * production.cloudflare.docker.com
* To report operational status
  * healthchecks.verbis.dkfz.de
* Beam
  * Beam broker URL

> 📝 Ubuntu's pre-installed uncomplicated firewall (ufw) is known to conflict with Docker, more info [here](https://github.com/chaifeng/ufw-docker).

## Deployment

### Base Installation

Clone this repository:
```shell
sudo mkdir -p /srv/docker/
sudo git clone https://github.com/samply/ecdc_central_server.git /srv/docker/ecdc_central_server
cd
git clone https://github.com/samply/lens.git
cd lens
git checkout ehds2
vi packages/demo/src/AppECDC.svelte # Search for "spot" and replace URL with the one at your site
docker build -t samply/lens --no-cache .
cd /srv/docker/ecdc_central_server
sudo mkdir -p letsencrypt conf/pki
sudo vi .env # Modify to set correct values of Lens and Spot endpoints
```

### Register with Samply.Beam

You will need to enroll with your Beam, using te [enroll software](https://github.com/samply/beam-enroll).

``` shell
cd /srv/docker/ecdc_central_server
sudo docker run --rm -ti -v ./conf/pki:/etc/bridgehead/pki samply/beam-enroll:latest --output-file $PRIVATEKEYFILENAME --proxy-id $PROXY_ID
```

Instead, log on to the VM where Beam is running and perform the following (you will need root permissions):
```shell
cd /srv/docker/beam-broker
sudo mkdir -p csr
sudo vi csr/ecdc-locator.csr # Copy and paste the certificate printed during the enroll
sudo pki-scripts/managepki sign --csr-file csr/ecdc-locator.csr --common-name=ecdc-locator.broker.bbmri.samply.de
```

You can check that the Locator has connected to Beam with the following command:
```shell
pki-scripts/managepki list

```

### Starting and stopping

To start, run

```shell
cd /srv/docker/ecdc_central_server
docker-compose up -d
```

To stop, run

```shell
cd /srv/docker/ecdc_central_server
docker-compose down
```

Once the components are running, you can also view the individual Docker processes with:

```shell
docker ps
```

There should be 3 Docker proceses. If there are fewer, then you know that something has gone wrong. To see what is going on, run:

Once the components have passed these checks, take a look at the landing page:

```
https://localhost
```

You can either do this in a browser or with curl. If you visit the URL in the browser, you will neet to click through several warnings, because you will initially be using a self-signed certificate. With curl, you can bypass these checks:

```shell
curl -k https://localhost
```

If you get errors when you do this, you need to use ```docker logs``` to examine your landing page container in order to determine what is going wrong.

## Site-specific configuration

### Non-Linux OS

The installation procedures described above have only been tested under Linux.

Below are some suggestions for getting the installation to work on other operating systems. Note that we are not able to provide support for these routes!

We believe that it is likely that installation would also work with FreeBSD and MacOS.

Under Windows, you have 2 options:

- Virtual machine
- WSL

We have tested the installation procedure with an Ubuntu 22.04 guest system running on WSL. That worked flawlessly for HTTP, ran into CORS problems with HTTPS.

## Troubleshooting

### Docker Daemon Proxy Configuration

Docker has a background daemon, responsible for downloading images and starting them. Sometimes, proxy configuration from your system won't carry over and it will fail to download images. In that case, you'll need to configure the proxy inside the system unit of docker by creating the file `/etc/systemd/system/docker.service.d/proxy.conf` with the following content:

``` ini
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=https://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,some-local-docker-registry.example.com,.corp"
```

After saving the configuration file, you'll need to reload the system daemon for the changes to take effect:

``` shell
sudo systemctl daemon-reload
```

and restart the docker daemon:

``` shell
sudo systemctl restart docker
```

For more information, please consult the [official documentation](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy).

## License

Copyright 2019 - 2024 The Samply Community

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
