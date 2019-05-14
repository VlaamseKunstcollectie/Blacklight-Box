# Blacklight Box (Vagrant, Packer and Ansible)

[![License: GPL v3](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](http://www.gnu.org/licenses/gpl-3.0)

This project builds and configures a virtual image containing the necessary
dependencies for developing, deploying and managing 
[Project Blacklight](https://github.com/projectblacklight/blacklight) instances on local 
or remote hosts.

You'll get an [Ubuntu 16.04.6 Server (386i)](http://releases.ubuntu.com/16.04/ubuntu-16.04.6-server-i386.iso) box ready for use with [Vagrant](https://www.vagrantup.com/). This box is provisioned with [Ansible](https://www.ansible.com/).

Provisioning from scratch can take a while, so [Packer](http://www.packer.io/)
support is included to generate a provisioned base box which you can reuse each
time you destroy an instance.

## Requirements

The following software must be installed/present on your local machine before
you can use Packer to build the Vagrant box file:

  - [Vagrant](http://vagrantup.com/)
  - [VirtualBox](https://www.virtualbox.org/)
  - [Packer](http://www.packer.io/) (optional)
  - [Ansible](http://docs.ansible.com/intro_installation.html) (optional)

## Quickstart

Start by configuring your box. Copy the `default.config.yml` file to a new
`config.yml` file. Settings in `config.yml` override the default settings as 
set in `default.config.yml`.

```bash
$ cp default.config.yml config.yml
```

Configure the `local_path` key of the `vagrant_synced_folders` setting. This 
should point to a folder on your host machine. The folder will be shared with 
the guest machine through the `/vagrant` folder.

If your setup looks like this and you want to share the Projects folder:

```bash
$ cd ~/Projects
$ ls -lah
drwxr-xr-x  12 user  staff   408B Oct 24 11:09 .
drwxr-xr-x  58 user  staff   1.9K Dec  2 16:50 ..
drwxr-xr-x  27 user  staff   918B Oct 24 15:59 project-foobar
```

the YAML configuration should look like this:

```bash
vagrant_synced_folders:
  # The first synced folder will be used for the default Datahub installation, if
  # any of the build_* settings are 'true'. By default the folder is set to
  # the datahub folder.
  - local_path: /Users/username/Projects
    destination: /vagrant
    type: nfs
create: true
```

Save and close the file. Next, boot the machine.

```bash
$ vagrant up
````

This will download and install a box file from [Vagrant Cloud](https://app.vagrantup.com/) 
that contains all the dependencies you need to run Project Blacklight 7.x

Booting should end with a status message:

```bash
==> blacklight.box: Machine 'blacklight.box' has a post `vagrant up` message. This is a message
==> blacklight.box: from the creator of the Vagrantfile, and not from Vagrant itself:
==> blacklight.box:
==> blacklight.box: Your Blacklight VM Vagrant box is ready to use!
==> blacklight.box:  Visit https://github.com/projectblacklight/blacklight for instructions on how to
==> blacklight.box:  install a new instance of Project Blacklight.
```

Now, SSH into your new box, install the Rails and Solrwrapper gems, and finish by
installing a new Project Blacklight instance.

```bash
$ vagrant ssh
$ gem install rails
$ gem install solr_wrapper
$ cd /vagrant
$ rails new search_app -m https://raw.github.com/projectblacklight/blacklight/master/template.demo.rb
```

This will create a new folder called `demo` in your vagrant folder which will be 
accessible from your host machine (OSX, Windows,...).

Now, edit the `.solr_wrapper.yml` file in the `demo` folder. Add the version like this:

```yml
# Place any default configuration for solr_wrapper here
# port: 8983
version: 7.3.1
collection:
  dir: solr/conf/
  name: blacklight-core
```

Next up, start the Rails server in daemon mode. And finish by starting Apache solr.

```bash
$ rails s -d
$ solr_wrapper
```

Go to your browser, and point it to [http://blacklight.box](http://blacklight.box).
You should be greeted by the home page of a vanilla Project Blacklight installation.

## Provisioning

The box can be altered by changing the Ansible configuration in the `ansible` folder
and running `ansible-playbook`. 

Copy or rename the `blacklight.default` file which contains the Ansible 
configuration file:

```bash
$ cp ./ansible/blacklight.default ./ansible/blacklight
```

Change the `path-to-machine` string in the inventory configuration and let it
point to the directory that contains the blacklight-Box installation i.e:

```
ansible_ssh_private_key_file='/Users/John/Machines/My-Box/.vagrant/machines/blacklight.box/virtualbox/private_key'
```

Next, make sure you have all the required software (listed above) before is installed,
then cd into the `ansible/extension/setup` directory and run:

```bash
$ sh role_update.sh
```

This script will pull down a set of external ansible roles created and
maintained by other authors. These are installed in the `ansible/roles/external`
directory.

Make changes to the Ansible roles and playbook accordingly. Then run the playbook:

```bash
$ cd ansible/plays
$ ansible-playbook -i ../blacklight blacklight.yml
```

## Building a box from scratch

It's possible to create a custom box file from scratch and use that instead of 
downloading a box file from Vagrant cloud.

Make sure you have completed the steps from the provisioning section. Then run
the packer command from the packer folder:

```bash
$ cd packer
$ packer build blacklight.json
```

After a few minutes, Packer should tell you the box was generated succesfully.
You'll find the box file in the `packer/build` directory.

Adjust the configuration in your `config.yml` file, making Vagrant use the 
local box file instead of downloading a new box as defined in `default.config.yml`:

```yml
vagrant_box: blacklight-xenial32
vagrant_box_url: file://packer/build/blacklight-xenial32.box
```

### Networking

Vagrant will automatically update your `/etc/hosts` file with the correct 
entries. If this hasn't happened, append these lines to your `/etc/hosts` file:

```bash
192.168.2.160   blacklight.box   # http://blacklight.box
```

Access:

| URL                         |  Destination                       |
| --------------------------- |  --------------------------------- |
|  http://blacklight.box      |  Your Project Blacklight instance  |
|  http://blacklight.box:3000 |  Direct access to Rails server     |
|  http://blacklight.box:8983 |  Direct access to Solr             |

You can add new domains and serve multiple instances of Project Blacklight. 
This requires extra steps:

* Add a new domain in the `config.yml` file under the domains and `nginx_hosts` sections
* Edit the `ansible\group_vars\all\nginx.yml` file, add a new vhost configuration block 
  and provision the box again through `ansible-playbook`
* Make sure that you point to a different port then 3000 (i.e 3001) to avoid conflicts 
  with already defined vhosts. When starting `rails` add the `-p 3001` switch.

You can add or change the IP address in the `config.yml` file.

## Authors

* Matthias Vandermaesen <matthias.vandermaesen@vlaamsekunstcollectie.be>

## Copyright

Copyright 2019 - Vlaamse Kunstcollectie vzw

## License

This library is free software; you can redistribute it and/or modify it under
the terms of the GPLv3.
