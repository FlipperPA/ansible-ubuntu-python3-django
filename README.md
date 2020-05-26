# Ubuntu 18.04, Python 3 & Django Ansible Playbooks and Roles

This repository holds Ansible playbooks and roles for configuring Python 3 & Django servers. You should be able to clone the repository and follow these steps to get a server up and running; this has been tested with Digital Ocean droplets, and can be adapted to other infrastructures as well.

These instructions assume the following:

* You have a working knowledge of [how SSH keypairs work](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys).
* You have a Django project to deploy. You can deploy it with a Django app such as [Django SSH Deployer](https://github.com/FlipperPA/django-ssh-deployer).

## Preparing the Destination Server

The destination server will need to be kickstarted with Ubuntu 18.04. Log in as root to your freshly created Digital Ocean droplet (you used SSH keys, right?), and create a user on the destination server. I'll call this service account user `ansible_deploy`, but feel free to name it something different. If you name it something different, you'll want to change it in `ansible.cfg` as well.

```bash
adduser --disabled-password --gecos "" ansible_deploy
usermod -aG sudo ansible_deploy
chmod 640 /etc/sudoers
```

Then edit `/etc/sudoers`. Find this line:
`%sudo   ALL=(ALL:ALL) ALL`
...and change it to:
`%sudo   ALL=(ALL:ALL) NOPASSWD: ALL`

Then let's become the `ansible_deploy` user, and generate keys:

```bash
su - ansible_deploy
ssh-keygen -b 4096
```

After issuing the `ssh-keygen` command, hit enter three times to use the defaults. You will then need to add your public key from your host control machine to `ansible_deploy`'s authorized keys in `~/.ssh/authorized_keys`:

```bash
echo "ssh-ed25519 AAAADummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDu you@yourdomain.com" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
exit
```

We'll want to create a separate service user account with less privileges to deploy your Django project. I'll call this account `web_deploy`, but you can choose whatever name you like. The `echo` command will add your public key from your host control machine, just like it did for `ansible_deploy`.

```bash
adduser web_deploy
usermod -aG www-data web_deploy
su - web_deploy
ssh-keygen -b 4096
ssh-keygen -t ed25519
echo "ssh-ed25519 AAAADummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDu you@yourdomain.com" >> .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
cat .ssh/id_ed25519.pub
```

Then copy your `id_ed25519.pub` key as a deployment key. Under GitHub, this is found under your repository, `Settings`, `Deploy Keys`. After adding the key, make sure you can clone your repository to your home directory. This will also allow you to add your version control host's RSA key fingerprint, required the first time you connect.

## Getting started

Clone this repository.

    git clone git@github.com:YourUsername/ansible-ubuntu18-python3-django.git
    cd ansible-ubuntu18-python3-django.git

Edit the `inventory` file and add the destination servers.

    [web_py]
    dev.yourserver.com
    123.34.56.789

Edit the `ansible.cfg` file and change the `remote_user` to the one you created above on the destination server.

    remote_user = ansible_deploy

Deploy the server:

    ansible-playbook playbooks/web-py.yml

NOTE: if you're using Vagrant, you may get an error about the config file being world-writable due to Vagrant's shared folders. You can issue commands with an environment variable as a work-around:

    ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/web-py.yml

Show some servers:

    ansible ubuntu_18 --list-hosts
    ansible web_py --list-hosts

Run some commands; `shell` is required for arguments and shell functions (`command` runs without the target user's shell):

    ansible all -m command -a uptime -i inventory
    ansible all -m shell -oa 'ps -eaf'
    ansible ubuntu_18 -m copy -a 'content="Welcome to your server, Ansiblized.\n" dest=/etc/motd' -u apache --become --become-user root

Documentation: list help section, show documentation for specific command (with examples!):

    ansible-doc -l
    ansible-doc yum
