---
layout: post
title:  "Windows WSL development setup for Python and Docker"
date:   2020-04-13 19:00:00 -0400
categories: [windows, wsl]
tags: [windows, wsl, linux, docker, python]
toc: true
---
In the last few months, I've decided to switch my home computer from a Mac to a PC. The reasoning behind it was pretty simple: I wanted to game a little more and most of the games I like just work better on Windows. That being said, the geek in me still wants to do some development sometimes and play with technologies that are more suited for Linux and for *nix based OSes.

That being said, on that PC, I've had a dual-boot with Linux installed but I was wondering what the options were on Windows 10 these days.

Install and configure Windows Sub-system for Linux (WSL)
========================================================================

This is well known stuff and there's a bunch of well made articles about it. Personally, I've followed Microsoft's documentation  [here](https://docs.microsoft.com/en-us/windows/wsl/install-win10). At the time of writing this, I'm still waiting for WSL2 to come out of development. If rumors are right, this should happen in May 2020.

I'll document the steps I took and some of the tweaks I made to have it work for my setup.

1. In powershell as administrator

```shell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

2. Reboot the system when this step finishes and prompted.

3. Install your favorite distribution from the Microsoft store. Personnally, I've used [Debian](https://www.microsoft.com/store/apps/9MSVKQC78PK6). Instructions will be similar as Ubuntu.

4. Initialize your distro as explained [here](https://docs.microsoft.com/en-us/windows/wsl/initialize-distro).

5. Make sure the distro is up to date.

```shell
sudo apt update && sudo apt upgrade
```

6. Create a WSL configuration that will make it compatible with Docker and development. This will switch the mounts of the Windows volumes from `/mnt/c` to `/c` and so on.

```shell
sudo cat >/etc/wsl.conf <<EOF
# Enable extra metadata options by default
[automount]
enabled = true
root = /
options = "metadata,umask=22,fmask=11"
mountFsTab = false

# Enable DNS – even though these are turned on by default, we’ll specify here just to be explicit.
[network]
generateHosts = true
generateResolvConf = true
EOF
```

Install and configure Docker for Windows
========================================================================

This part will be expected to change in the future when WSL2 comes out. For now, we'll stick with having Docker in its seperate VM.

Installation instructions are found [here](https://docs.docker.com/docker-for-windows/install/). In summary:

1. Download the package from [Docker hub](https://hub.docker.com/editions/community/docker-ce-desktop-windows/) and install it.

2. Go to [Docker Settings](https://docs.docker.com/docker-for-windows/#docker-settings-dialog) and check the "Expose daemon on tcp://localhost:2375 without TLS" option. Then click on "Apply and restart".

Install and configure Windows Terminal
========================================================================

On Windows, we need a proper shell. In the free category, the closest I found to a proper terminal on Windows is [Windows Terminal](https://github.com/microsoft/terminal). This is a project from Microsoft which is still in preview but for now works quite well. I've started to play with it and I've only started to customize it to my taste. Here's what I did:

1. Get the it from the [Microsoft Store](https://aka.ms/windowsterminal) and install it. This will ensure you get updates.

2. Go to the "Settings" window by pressing `Ctrl-,`.

3. Change the `defautProfile` value for the WSL `guid` value.

4. Add the `initialCols` with a value of `180` to the main map.

5. Add the `initialRows` with a value of `40` to the main map.

6. Add the `initcopyOnSelectialCols` with a value of `true` to the main map.

7. In the `defaults` map, add the `colorScheme` with the value `"One Half Dark"` and the `fontFace` attribute to `"Operator Mono"`. 

You should end up with a settings JSON that looks like this:

```json
{
    "$schema": "https://aka.ms/terminal-profiles-schema",

    "defaultProfile": "{58ad8b0c-3ef8-5f4d-bc6f-13e4c00f2530}",
    "initialCols": 180,
    "initialRows": 40,
    "copyOnSelect": true,
    "profiles":
    {
        "defaults":
        {
            "padding": "10, 10, 10, 10",
            "fontFace": "Operator Mono",
            "colorScheme": "One Half Dark"
        },
        "list":
        [
            {
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "name": "Windows PowerShell",
                "commandline": "powershell.exe",
                "hidden": false
            },
            {
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "name": "cmd",
                "commandline": "cmd.exe",
                "hidden": false
            },
            {
                "guid": "{58ad8b0c-3ef8-5f4d-bc6f-13e4c00f2530}",
                "hidden": false,
                "name": "Debian",
                "source": "Windows.Terminal.Wsl"
            },
            {
                "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
                "hidden": false,
                "name": "Azure Cloud Shell",
                "source": "Windows.Terminal.Azure"
            },
            {
                "guid": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",
                "hidden": false,
                "name": "PowerShell",
                "source": "Windows.Terminal.PowershellCore"
            }
        ]
    },

    "schemes": [],

    "keybindings": []
}
```

Install and configure zsh, Docker and others in WSL.
========================================================================

I'm starting to get used to zsh since it's now the new default shell on MacOS so I'll set it up and experiment on this setup.

1. Start a Windows Terminal.

2. Install zsh, direnv and pre-requisites to [Oh My zsh](https://github.com/ohmyzsh/ohmyzsh).
```shell
sudo apt-get update && sudo apt-get install zsh curl wget git direnv
```

3. Install via `curl`
```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

4. Edit your `~/.zshrc` and setup your preferences.
```shell
vi ~/.zshrc
```
    Personnaly, I've setup the following options
```shell
ZSH_THEME="maran"
CASE_SENSITIVE="true"
plugins=(git debian)
```

    In the `User configuration` section, I did the following:
```shell
export PATH=$HOME/bin:$HOME/.local/bin:$PATH
# Docker engine
export DOCKER_HOST=tcp://127.0.0.1:2375
# AWS Variables
source $HOME/.aws/env.sh
# Add an SSH agent.
eval $(ssh-agent) > /dev/null
# Hook up direnv
eval "$(direnv hook zsh)"
# Ruby Crap
# Install Ruby Gems to ~/gems
export GEM_HOME="$HOME/gems"
export PATH="$HOME/gems/bin:$PATH"
```

    This will:
    - Add the local bin to the path. Also if you do a `mkdir -p $HOME/bin`, you can put binaries in there.
    - Set `DOCKER_HOST` to use the TCP socket we opened previously.
    - Create an SSH agent with your local keys.
    - Enable `direnv`.
    - Since I'm using Jekyll for this blog. Install the Ruby stuff. `mkdir -p $HOME/gems` would help.

5. Exit and restart Windows Terminal and see your changes.

6. Install Docker in the WSL Debian. We will follow instructions in [here](https://docs.docker.com/engine/install/debian/)

    - Install pre-requisites
```shell
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

    - Add the GPG key.
```shell
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
    - Add the repository
```shell
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```
    - Install the packages.
```shell
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```
    - Add your user to the `docker` group.
```shell
sudo usermod -aG docker $USER
```
    - Test docker
```shell
docker run hello-world
```

Setup Python in WSL
========================================================================

1. Install Python 3.
```shell
sudo apt-get update && sudo apt-get install build-essential python3.7 python3.7-dev python3-pip
```

2. Setup links for `python` and `pip`
```shell
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3 1
sudo update-alternatives --install /usr/bin/pip pip /usr/bin/pip3 1
```

3. Update `pip`
```shell
sudo pip install --upgrade pip
```

4. Install `pipenv`
```shell
pip install --user pipenv
```

5. See that `pipenv` is working properly.
```shell
pipenv --version
```

Install and configure Visual Studio Code
========================================================================

The first IDE I'm planning on testing is Visual Studio Code. It has a nice integration with WSL that's free. Here's how to install and do a basic configuration for it:

1. [Download](https://code.visualstudio.com/download) Visual Studio Code.

2. Follow the setup instructions [here](https://code.visualstudio.com/docs/setup/setup-overview).

3. Install the remote components by follow the instructions [here](https://code.visualstudio.com/remote-tutorials/wsl/getting-started).

4. Since I'm going to follow the [Python instructions](https://code.visualstudio.com/docs/languages/python).

5. We can now issue `code .` while in a directory in Windows Terminal and this will pop-up Visual Studio Code in WSL.

That's all folks!
