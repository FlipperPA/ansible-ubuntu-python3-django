# Ubuntu, Python 3, Apache & Django: Ansible Playbooks and Roles

We could call this something moronic like the U-PAD stack, but just in case you're ever required to use this particular concoction of technology, hopefully there are some helpful notes and examples here. This repository holds Ansible playbooks and roles for configuring Python 3 & Django servers; you should be able to clone the repository and follow these steps to get a server up and running; this has been tested with Digital Ocean droplets, and can be adapted to other infrastructures as well.

These instructions assume the following:

* You have a working knowledge of [how SSH keypairs work](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys).
* You have a Django project to deploy. You can deploy it with a Django app such as [Django SSH Deployer](https://github.com/FlipperPA/django-ssh-deployer).

## Preparing the Destination Server

The destination server will need to be kickstarted with Ubuntu; be sure [you've uploaded your public key to Digital Ocean or whatever ISP you're using, and included it during droplet creation](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/). Be sure to copy the commands below line-by-line; some command require interaction, and if you're learning, it will help you figure out what each command does.

### Creating a Service User for Ansible

* First, log in as root to your freshly created Digital Ocean droplet
* Then, create a service account to run Ansible. I'll call this service account user `ansible_deploy`, but feel free to name it something different. If you name it something different, you'll want to change it in `ansible.cfg` as well. Here are the command to create the user:

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

After issuing the `ssh-keygen` command, hit enter three times to use the defaults. You will then need to add your public key from your host control machine to `ansible_deploy`'s authorized keys in `~/.ssh/authorized_keys`. In the following examples, replace `ssh-ed25519 AAAADummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDu you@yourdomain.com` with your key from your local machine. This key is often found under your home directory in the `.ssh` sub-directory: `~/.ssh/id_ed25519.pub` or `~/.ssh/id_rsa.pub`.

```bash
echo "ssh-ed25519 AAAADummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDu you@yourdomain.com" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
exit
```

### Create a Service User for Deploying Your Web Code

We'll want to create a separate service user account with less privileges to deploy your Django project. I'll call this account `web_deploy`, but you can choose whatever name you like. The `echo` command will add your public key from your host control machine, just like it did for `ansible_deploy`.

```bash
adduser --disabled-password --gecos "" web_deploy
usermod -aG www-data web_deploy
su - web_deploy
ssh-keygen -b 4096
ssh-keygen -t ed25519
echo "ssh-ed25519 AAAADummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDummyDu you@yourdomain.com" >> .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
cat .ssh/id_ed25519.pub
```

Then copy your `id_ed25519.pub` key as a deployment key. Under GitHub, this is found under your repository, `Settings`, `Deploy Keys`. After adding the key, make sure you can clone your repository to your home directory. This will also allow you to add your version control host's RSA key fingerprint, required the first time you connect.

### Testing the Service Accounts

Now, let's ensure we can connect as our service accounts from our control machine, and that our web deployment account can connect to our code repository. In this example, I'll test whether we can connect to GitHub.

```bash
# Connect to your Digital Ocean Droplet server as the Ansible service user
ssh ansible_deploy@yourserver.com
# Exit back to your control machine
exit
# Connect to your Digital Ocean Droplet server as the web service user
ssh web_deploy@yourserver.com
# Connect from your Digital Ocean Droplet server to GitHub to test your deployment key
ssh git@github.com
```

You should see something like this; type `yes` to add the key permanently:

```bash
web_deploy@ubuntu-django-prod-1:~$ ssh git@github.com
The authenticity of host 'github.com (140.82.114.4)' can't be established.
RSA key fingerprint is SHA256:DummyDummyDummyDummyDummyDummyDummyDummyDum.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'github.com' (RSA) to the list of known hosts.
PTY allocation request failed on channel 0
Hi YourUsername/YourSite! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

Your server is now ready to go! We can now built the server with Ansible and deploy our code.

## Getting started

Clone this repository.

    git clone git@github.com:YourUsername/ansible-ubuntu-python3-django.git
    cd ansible-ubuntu-python3-django.git

Edit the `inventory` file and add the destination servers.

    [web_py]
    dev.yourserver.com
    123.34.56.789

Edit the `ansible.cfg` file and change the `remote_user` to the one you created above on the destination server.

    remote_user = ansible_deploy

Deploy the server:

    ansible-playbook playbooks/web-py.yml

Show some servers:

    ansible ubuntu --list-hosts
    ansible web_py --list-hosts

Run some commands; `shell` is required for arguments and shell functions (`command` runs without the target user's shell):

    ansible all -m command -a uptime -i inventory
    ansible all -m shell -oa 'ps -eaf'
    ansible ubuntu -m copy -a 'content="Welcome to your server, Ansiblized.\n" dest=/etc/motd' -u apache --become --become-user root

Documentation: list help section, show documentation for specific command (with examples!):

    ansible-doc -l
    ansible-doc yum

## Notes for Apache

Most folks these days run gunicorn with nginx, but these examples are for Apache. We've had to do a few extra things for Apache to run as a Reverse Proxy with gunicorn for Django:

* Ensure that you've set `USE_X_FORWARDED_HOST = True` and `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')` in your project's Django settings, or it will return `http://127.0.0.1/` when URLs are built by `request.build_absolute_uri` instead of your domain.
* We've set `RequestHeader set X_FORWARDED_PROTO 'https' env=HTTPS` and `RequestHeader set X-Forwarded-Ssl on` in Apache's configuration to properly pass through actual FQDN through the reverse proxy.
* To run as a service user, we've installed and activated the `libapache2-mpm-itk` module which allows use to set the `AssignUserId` directive in the Apache `VirtualHost` configuration.
