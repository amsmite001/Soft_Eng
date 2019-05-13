# Introduction

We cover how to set up JupyterHub in this document including:

- [Introduction](#introduction)
  - [Pre-requisites](#pre-requisites)
  - [Installation](#installation)
  - [Nginx-Configuration](#nginx-configuration)
  - [Jupyter-Configuration](#jupyter-configuration)
  - [JAAS](#jaas)
  - [OAuth](#oauth)
  - [Customization](#customization)
  - [Running Jupyter](#running-jupyter)

## Pre-requisites

* We **strongly reccomend using `Anaconda`** as your base system, but a Linux/Unix system at the very least is required.
* Python 3.5 or greater
* Nodejs/npm
  * This is default installed in Anaconda
  * Otherwise, install with:

  ``` (bash)
  sudo apt-get install npm nodejs-legacy
  ```

* A domain name

---  

## Installation

To begin the installation of Jupyter, ensure that the system is up to date with

``` (bash)
apt update
apt upgrade
```

Now, use the system package manager to handle the install of Jupyter Notebook & JupyterHub
<details open>
<summary>Conda</summary>
<br>

``` (bash)
conda install -c conda-forge jupyterhub
conda install notebook
```

</details>


<details>
<summary>Other Unix</summary>
<br>

``` (bash)
python3 -m pip install jupyterhub
npm install -g configurable-http-proxy
python3 -m pip install notebook
```

</details>

To reach a functional point, let DO have access to port 8000 with

``` (bash)
sudo ufw allow 8000
```

The server is now runnable with

``` (bash)
jupyterhub --no-ssl
```

You can access the server with `https://localhost:8000`.

---

## Nginx-Configuration

This is to configure the security keys for Jupyterhub and SSL:

``` (bash)
cd /srv
sudo mkdir jupyterhub
cd jupyterhub
sudo touch jupyterhub_cookie_secret
sudo chown :sudo jupyterhub_cookie_secret
sudo chmod g+rw jupyterhub_cookie_secret
sudo openssl rand -hex 32 > jupyterhub_cookie_secret
sudo chmod 600 jupyterhub_cookie_secret
sudo touch proxy_auth_token
sudo chown :sudo proxy_auth_token
sudo chmod g+rw proxy_auth_token
sudo openssl rand -hex 32 > proxy_auth_token
sudo chmod 600 proxy_auth_token
sudo touch dhparam.pem
sudo chown :sudo dhparam.pem
sudo chmod g+rw dhparam.pem
sudo openssl dhparam.pem -out /srv/jupyterhub/dhparam.pem 2048
sudo chmod 600 dhparam.pem
```

## Jupyter-Configuration

The default `jupyterhub_config.py` file contains comments and guidance for all configuration variables and their default values. To generate the config file, it's reccomended to be in the home directory.

``` (bash)
jupyterhub --generate-config
```

Replace (or ensure that the rest of the config file is commentedout) the contents with

``` (python)
import git, os, shutil
from pwd import getpwnam
from git import Repo

c = get_config()
c.JupyterHub.log_level = 10

#Spawn JupyterLab
c.Spawner.cmd = '/home/user/anaconda3/bin/jupyter-labhub'

#True means that each time a user logs in, the default Git repo will overwrite the local files
ERASE_DIR = False

#Github Oath authentication
from oauthenticator.github import LocalGitHubOAuthenticator
c.JupyterHub.authenticator_class = LocalGitHubOAuthenticator
c.LocalGitHubOAuthenticator.create_system_users = True
#Github credentials
c.LocalGitHubOAuthenticator.oauth_callback_url = 'https://notebooks.jupyterbin.com/hub/oauth_callback'
c.LocalGitHubOAuthenticator.client_id = 'd4aae9cc903709f55b06'
c.LocalGitHubOAuthenticator.client_secret = 'a237b7239d22a22c50e5b77d6df695572d024bfd'

#This is how the authenticator adds PAM credentials upon new user. 
#"--force-badname" allows usernames that start with a number, for example, so that all github usernames
#may be used
c.Authenticator.add_user_cmd = ['adduser', '-q', '--gecos', '""', '--disabled-password', '--force-badname']
# Cookie Secret Files
c.JupyterHub.cookie_secret_file = '/srv/jupyterhub/jupyterhub_cookie_secret'
c.ConfigurableHTTPProxy.auth_token = '/srv/jupyterhub/proxy_auth_token'
c.Authenticator.delete_invalid_users = True

#c.Authenticator.whitelist = {'user'}

#Make sure that your master PAM user and admin Github username are listed here for admin purposes
c.Authenticator.admin_users = {'user, [Git_Username]'}

#This pulls down specified Github and replicates it locally upon user creation
def create_dir_hook(spawner):
    """
    A function to clone a github repo into a specific directory of a 
   JupyterHub user when the server spawns a new notebook instance.
    """
    username = spawner.user.name
    DIR_NAME = os.path.join("/home", username)
    git_url = "https://github.com/2kreate/jbin.git"
    repo_dir = os.path.join(DIR_NAME, 'notebooks')
    uid1 = getpwnam(username).pw_uid
    gid1 = getpwnam(username).pw_gid

    if ERASE_DIR == True:
        if os.path.isdir(repo_dir):
            shutil.rmtree(repo_dir)
        os.mkdir(repo_dir)
        shutil.chown(repo_dir, user=uid1, group=gid1)
        clone_repo(username, git_url, repo_dir)

    if ERASE_DIR == False and not (os.path.isdir(repo_dir)):
        os.mkdir(repo_dir)
        shutil.chown(repo_dir, user=uid1, group=gid1)
        clone_repo(username, git_url, repo_dir)

    if ERASE_DIR == False and os.path.isdir(repo_dir):
        pass

def clone_repo(user, git_url, repo_dir):
    """
    A function to clone a github repo into a specific directory of a user.
    """
    git.Repo.clone_from(git_url, repo_dir)
    uid = getpwnam(user).pw_uid
    gid = getpwnam(user).pw_gid
    for root, dirs, files in os.walk(repo_dir):
        for d in dirs:
            shutil.chown(os.path.join(root, d), user=uid, group=gid)
        for f in files:
            shutil.chown(os.path.join(root, f), user=uid, group=gid)


c.Spawner.pre_spawn_hook = create_dir_hook
```

A few points to configure:

- Change `Git_Username` to the username of the individual whom you'd like to be admin.
- `git_url = "https://github.com/2kreate/jbin.git"` needs to be changed to whichever default repo you would like the users to start with.

## JAAS

To configure Jupyter as a Service, naviagte to

``` (bash)
cd /etc/systemd/system
vim jupyterhub.service
```

Add the following to the file just created. Save and quit.

```
[Unit]
Description=Jupyterhub
After=syslog.target network.target

[Service]
User=root
Environment="PATH=/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/home/user/anaconda3/bin"
ExecStart=/home/user/anaconda3/bin/jupyterhub -f /home/user/jupyterhub_config.py

[Install]
WantedBy=multi-user.target
```

**Important**: Afterwards run

``` (bash)
sudo systemctl daemon-reload
```

---

## OAuth

This configuration example is specific for Github Oathenticator

---

## Customization

Jupyter allows the customization of it's html files via templates. To use the provided template override, navigate to `anaconda3/share/jupyterhub/templates` and replace the `login.html` with the provided one.

Editing templates follows Jinja styling, and are just a matter of editing/replacing existing files.

---

## Running Jupyter

You can start/stop jupyterhub with

``` (bash)
sudo systemctl start jupyterhub
sudo systemctl stop jupyterhub
```

This will mimic a restart if necessary. You can see the server status with the `status` flag instead of `start/stop`.

sudo journalctl -u jupyterhub gives you access to the full Jupyterhub log.
