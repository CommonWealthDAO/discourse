<<<<<<< HEAD
# Discourse Docker images

## About

- [Docker](https://docker.com/) is an open source project to pack, ship and run any Linux application in a lighter weight, faster container than a traditional virtual machine.

- Docker makes it much easier to deploy [a Discourse forum](https://github.com/discourse/discourse) on your servers and keep it updated. For background, see [Sam's blog post](http://samsaffron.com/archive/2013/11/07/discourse-in-a-docker-container).

- The templates and base image configure Discourse with the Discourse team's recommended optimal defaults.

## Getting Started

The simplest way to get started is via the **standalone** template, which can be installed in 30 minutes or less. For detailed install instructions, see

https://github.com/discourse/discourse/blob/main/docs/INSTALL-cloud.md

## Directory Structure

### `/cids`

Contains container ids for currently running Docker containers. cids are Docker's "equivalent" of pids. Each container will have a unique git like hash.

### `/containers`

This directory is for container definitions for your various Discourse containers. You are in charge of this directory, it ships empty.

### `/samples`

Sample container definitions you may use to bootstrap your environment. You can copy templates from here into the containers directory.

### `/shared`

Placeholder spot for shared volumes with various Discourse containers. You may elect to store certain persistent information outside of a container, in our case we keep various logfiles and upload directory outside. This allows you to rebuild containers easily without losing important information. Keeping uploads outside of the container allows you to share them between multiple web instances.

### `/templates`

[pups](https://github.com/discourse/pups)-managed templates you may use to bootstrap your environment.

### `/image`

Dockerfiles for Discourse; see [the README](image/README.md) for further details.

The Docker repository will always contain the latest built version at: https://hub.docker.com/r/discourse/base/, you should not need to build the base image.

## Launcher

The base directory contains a single bash script which is used to manage containers. You can use it to "bootstrap" a new container, enter, start, stop and destroy a container.

```
Usage: launcher COMMAND CONFIG [--skip-prereqs] [--docker-args STRING]
Commands:
    start:       Start/initialize a container
    stop:        Stop a running container
    restart:     Restart a container
    destroy:     Stop and remove a container
    enter:       Open a shell to run commands inside the container
    logs:        View the Docker logs for a container
    bootstrap:   Bootstrap a container for the config based on a template
    run:         Run the given command with the config in the context of the last bootstrapped image
    rebuild:     Rebuild a container (destroy old, bootstrap, start new)
    cleanup:     Remove all containers that have stopped for > 24 hours
    start-cmd:   Generate docker command used to start container
```

If the environment variable "SUPERVISED" is set to true, the container won't be detached, allowing a process monitoring tool to manage the restart behaviour of the container.

## Container Configuration

The beginning of the container definition can contain the following "special" sections:

### templates:

```yaml
templates:
  - 'templates/cron.template.yml'
  - 'templates/postgres.template.yml'
```

This template is "composed" out of all these child templates, this allows for a very flexible configuration structure. Furthermore you may add specific hooks that extend the templates you reference.

### expose:

```yaml
expose:
  - '2222:22'
  - '127.0.0.1:20080:80'
```

Publish port 22 inside the container on port 2222 on ALL local host interfaces. In order to bind to only one interface, you may specify the host's IP address as `([<host_interface>:[host_port]])|(<host_port>):<container_port>[/udp]` as defined in the [docker port binding documentation](http://docs.docker.com/userguide/dockerlinks/). To expose a port without publishing it, specify only the port number (e.g., `80`).

### volumes:

```yaml
volumes:
  - volume:
    host: /var/discourse/shared
    guest: /shared
```

Expose a directory inside the host to the container.

### links:

```yaml
links:
  - link:
    name: postgres
    alias: postgres
```

Links another container to the current container. This will add `--link postgres:postgres`
to the options when running the container.

### environment variables:

Setting environment variables to the current container.

```yaml
env:
  DISCOURSE_DB_HOST: some-host
  DISCOURSE_DB_NAME: '{{config}}_discourse'
```

The above will add `-e DISCOURSE_DB_HOST=some-host -e DISCOURSE_DB_NAME=app_discourse` to the options when running the container.

### labels:

```yaml
labels:
  monitor: 'true'
  app_name: '{{config}}_discourse'
```

Add labels to the current container. The above will add `--l monitor=true -l app_name=dev_discourse` to the options
when running the container

## Upgrading Discourse

The Docker setup gives you multiple upgrade options:

1. Use the front end at http://yoursite.com/admin/upgrade to upgrade an already running image.

2. Create a new base image manually by running:
   `./launcher rebuild my_image`

## Single Container vs. Multiple Containers

The samples directory contains a standalone template. This template bundles all of the software required to run Discourse into a single container. The advantage is that it is easy.

The multiple container configuration setup is far more flexible and robust, however it is also more complicated to set up. A multiple container setup allows you to:

- Minimize downtime when upgrading to new versions of Discourse. You can bootstrap new web processes while your site is running and only after it is built, switch the new image in.
- Scale your forum to multiple servers.
- Add servers for redundancy.
- Have some required services (e.g. the database) run on beefier hardware.

If you want a multiple container setup, see the `data.yml` and `web_only.yml` templates in the samples directory. To ease this process, `launcher` will inject an env var called `DISCOURSE_HOST_IP` which will be available inside the image.

WARNING: In a multiple container configuration, _make sure_ you setup iptables or some other firewall to protect various ports (for postgres/redis).
On Ubuntu, install the `ufw` or `iptables-persistent` package to manage firewall rules.

## Email

For a Discourse instance to function properly Email must be set up. Use the `SMTP_URL` env var to set your SMTP address, see sample templates for an example. The Docker image does not contain postfix, exim or another MTA, it was omitted because it is very tricky to set up correctly.

## Troubleshooting

View the container logs: `./launcher logs my_container`

Spawn a shell inside your container using `./launcher enter my_container`. This is the most foolproof method if you have host root access.

If you see network errors trying to retrieve code from `github.com` or `rubygems.org` try again - sometimes there are temporary interruptions and a retry is all it takes.

Behind a proxy network with no direct access to the Internet? Add proxy information to the container environment by adding to the existing `env` block in the `container.yml` file:

```yaml
env:
  …existing entries…
  HTTP_PROXY: http://proxyserver:port/
  http_proxy: http://proxyserver:port/
  HTTPS_PROXY: http://proxyserver:port/
  https_proxy: http://proxyserver:port/
```

## Security

Directory permissions in Linux are UID/GID based, if your numeric IDs on the
host do not match the IDs in the guest, permissions will mismatch. On clean
installs you can ensure they are in sync by looking at `/etc/passwd` and
`/etc/group`, the Discourse account will have UID 1000.

## Advanced topics

- [Setting up SSL with Discourse Docker](https://meta.discourse.org/t/allowing-ssl-for-your-discourse-docker-setup/13847)
- [Multisite configuration with Docker](https://meta.discourse.org/t/multisite-configuration-with-docker/14084)
- [Linking containers for a multiple container setup](https://meta.discourse.org/t/linking-containers-for-a-multiple-container-setup/20867)
- [Using Rubygems mirror to improve connection problem in China](https://meta.discourse.org/t/replace-rubygems-org-with-taobao-mirror-to-resolve-network-error-in-china/21988/1)

## License

MIT
=======
<a href="https://www.discourse.org/">
  <img src="images/discourse-readme-logo.png" width="300px">
</a>

Discourse is the online home for your community. We offer a 100% open source community platform to those who want complete control over how and where their site is run.

Our platform has been battle-tested for over a decade and continues to evolve to meet users’ needs for a powerful community platform. Discourse allows you to create discussion topics and connect using real-time chat, as well as access an ever-growing number of official and community themes. In addition, we offer a wide variety of plugins for features ranging from chatbots powered by [Discourse AI](https://meta.discourse.org/t/discourse-ai/259214) to functionalities like SQL analysis using the [Data Explorer](https://meta.discourse.org/t/discourse-data-explorer/32566) plugin.

To learn more, visit [**discourse.org**](https://www.discourse.org) and join our support community at [meta.discourse.org](https://meta.discourse.org).

## Screenshots

<a href="https://blog.discourse.org/2023/08/discourse-3-1-is-here/"><img alt="Discourse 3.1" src="https://github-production-user-asset-6210df.s3.amazonaws.com/5862206/261215898-ae95f963-5ab4-4509-b87a-f9f6e9a109bf.png" width="720px"></a>

<a href="https://bbs.boingboing.net"><img alt="Boing Boing" src="https://github-production-user-asset-6210df.s3.amazonaws.com/5862206/261580781-1413ac96-5d08-40b2-bc8e-27c3f2d3bfe6.png" width="720px"></a>

<a href="https://twittercommunity.com/"><img alt="X Community" src="https://github.com/discourse/discourse/assets/2790986/ebb63eee-1927-4060-ada1-cf1bc774084c.png" width="720px"></a>

<img src="https://user-images.githubusercontent.com/1681963/52239118-b304f800-289b-11e9-9904-16450680d9ec.jpg" alt="Mobile" width="414">

Browse [lots more notable Discourse instances](https://www.discourse.org/customers).

## Development

To get your environment set up, follow the community setup guide for your operating system.

1. If you're on macOS, try the [macOS development guide](https://meta.discourse.org/t/beginners-guide-to-install-discourse-on-macos-for-development/15772).
1. If you're on Ubuntu, try the [Ubuntu development guide](https://meta.discourse.org/t/beginners-guide-to-install-discourse-on-ubuntu-for-development/14727).
1. If you're on Windows, try the [Windows 10 development guide](https://meta.discourse.org/t/beginners-guide-to-install-discourse-on-windows-10-for-development/75149).
1. If you're looking to use a simpler Docker-based install, try the [Docker development guide](https://meta.discourse.org/t/install-discourse-for-development-using-docker/102009).

If you're familiar with how Rails works and are comfortable setting up your own environment, you can also try out the [**Discourse Advanced Developer Guide**](docs/DEVELOPER-ADVANCED.md), which is aimed primarily at Ubuntu and macOS environments.

Before you get started, ensure you have the following minimum versions: [Ruby 3.2+](https://www.ruby-lang.org/en/downloads/), [PostgreSQL 13](https://www.postgresql.org/download/), [Redis 7](https://redis.io/download). If you're having trouble, please see our [**TROUBLESHOOTING GUIDE**](docs/TROUBLESHOOTING.md) first!

## Setting up Discourse

If you want to set up a Discourse forum for production use, see our [**Discourse Install Guide**](docs/INSTALL.md).

If you're looking for official hosting, see [discourse.org/pricing](https://www.discourse.org/pricing/).

## Requirements

Discourse is built for the *next* 10 years of the Internet, so our requirements are high.

Discourse supports the **latest, stable releases** of all major browsers and platforms:

| Browsers              | Tablets      | Phones       |
| --------------------- | ------------ | ------------ |
| Apple Safari          | iPadOS       | iOS          |
| Google Chrome         | Android      | Android      |
| Microsoft Edge        |              |              |
| Mozilla Firefox       |              |              |

Additionally, we aim to support Safari on iOS 15.7+.

## Built With

- [Ruby on Rails](https://github.com/rails/rails) &mdash; Our back end API is a Rails app. It responds to requests RESTfully in JSON.
- [Ember.js](https://github.com/emberjs/ember.js) &mdash; Our front end is an Ember.js app that communicates with the Rails API.
- [PostgreSQL](https://www.postgresql.org/) &mdash; Our main data store is in Postgres.
- [Redis](https://redis.io/) &mdash; We use Redis as a cache and for transient data.
- [BrowserStack](https://www.browserstack.com/) &mdash; We use BrowserStack to test on real devices and browsers.

Plus *lots* of Ruby Gems, a complete list of which is at [/main/Gemfile](https://github.com/discourse/discourse/blob/main/Gemfile).

## Contributing

[![Build Status](https://github.com/discourse/discourse/actions/workflows/tests.yml/badge.svg)](https://github.com/discourse/discourse/actions)

Discourse is **100% free** and **open source**. We encourage and support an active, healthy community that
accepts contributions from the public &ndash; including you!

Before contributing to Discourse:

1. Please read the complete mission statements on [**discourse.org**](https://www.discourse.org). Yes we actually believe this stuff; you should too.
2. Read and sign the [**Electronic Discourse Forums Contribution License Agreement**](https://www.discourse.org/cla).
3. Dig into [**CONTRIBUTING.MD**](CONTRIBUTING.md), which covers submitting bugs, requesting new features, preparing your code for a pull request, etc.
4. Always strive to collaborate [with mutual respect](https://github.com/discourse/discourse/blob/main/docs/code-of-conduct.md).
5. Not sure what to work on? [**We've got some ideas.**](https://meta.discourse.org/t/so-you-want-to-help-out-with-discourse/3823)


We look forward to seeing your pull requests!

## Security

We take security very seriously at Discourse; all our code is 100% open source and peer reviewed. Please read [our security guide](https://github.com/discourse/discourse/blob/main/docs/SECURITY.md) for an overview of security measures in Discourse, or if you wish to report a security issue.

## The Discourse Team

The original Discourse code contributors can be found in [**AUTHORS.MD**](docs/AUTHORS.md). For a complete list of the many individuals that contributed to the design and implementation of Discourse, please refer to [the official Discourse blog](https://blog.discourse.org/2013/02/the-discourse-team/) and [GitHub's list of contributors](https://github.com/discourse/discourse/contributors).

## Copyright / License

Copyright 2014 - 2023 Civilized Discourse Construction Kit, Inc.

Licensed under the GNU General Public License Version 2.0 (or later);
you may not use this work except in compliance with the License.
You may obtain a copy of the License in the LICENSE file, or at:

   https://www.gnu.org/licenses/old-licenses/gpl-2.0.txt

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

Discourse logo and “Discourse Forum” ®, Civilized Discourse Construction Kit, Inc.

## Accessibility

To guide our ongoing effort to build accessible software we follow the [W3C’s Web Content Accessibility Guidelines (WCAG)](https://www.w3.org/TR/WCAG21/). If you'd like to report an accessibility issue that makes it difficult for you to use Discourse, email accessibility@discourse.org. For more information visit [discourse.org/accessibility](https://discourse.org/accessibility).

## Dedication

Discourse is built with [love, Internet style.](https://www.youtube.com/watch?v=Xe1TZaElTAs)
>>>>>>> origin/main
