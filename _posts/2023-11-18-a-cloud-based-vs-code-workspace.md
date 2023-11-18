---
layout: post
title: "A cloud-based VS Code workspace"
date: "2023-11-18 16:25:00 -0300"
excerpt: Setting up a multi-repository VS Code workspace on GitHub Codespaces.
---

![VS Code running in a full-screen browser window, connected to a GitHub codespace, editing Python code for a graphical ERP application called [Oks](https://github.com/alnvdl/oks), with a terminal open and an embedded browser displaying the GTK interface of the application.](/assets/img/workspace_screenshot.png)
*VS Code running in a full-screen browser window, connected to a GitHub codespace, editing Python code for a graphical ERP application called [Oks](https://github.com/alnvdl/oks), with a terminal open and an embedded browser displaying the GTK interface of the application.*

This post will show how to set up a multi-repository VS Code workspace that runs entirely in the cloud, using [GitHub Codespaces](https://github.com/features/codespaces). It is meant as a guide for others who may find this idea interesting, and as a future reference for myself.

GitHub Codespaces lets you start a VM in the cloud and then [points a VS Code instance at it](https://code.visualstudio.com/docs/remote/remote-overview). To simplify the setup of that VM, everything actually runs within a Docker container. At the time of this writing, everyone gets up to 60 hours of codespace runtime for free every month on a 2-core instance with 8 GB of RAM.

It is an incredibly neat technology, and the free tier has been working very well for me. Given that I mostly develop backend and web frontend software without any serious physical hardware requirements, I'm seriously starting to rethink the types of computers I want on my desk in the future...

Codespaces enable two things that I find very valuable:
- Specifying your entire development environment as code.
- Start or continue development very fast, without being tied to a single machine.

Of course, both of these things could be done locally. But it's all so nicely built and integrated with GitHub that it's just too convenient to pass :-)

This post was written in a codespace. The entire code for my VS Code workspace running on GitHub codespaces is available at [https://github.com/alnvdl/workspace](https://github.com/alnvdl/workspace).

The sections below describe a few bits of knowledge I gathered while build this workspace.

## Creating the container image
By default, GitHub Codespaces starts a container from a [generic image](https://github.com/microsoft/vscode-dev-containers/tree/main/containers/codespaces-linux) full of development tools for many different programming languages.

I didn't need all of that. And given that the free tier storage is limited to 15 GB, it was wise to simplify that base image.

That is very easy to do using two files in a repo: `devcontainer.json` and `Dockefile`. The [Introduction to dev containers](https://docs.github.com/en/codespaces/setting-up-your-project-for-codespaces/adding-a-dev-container-configuration/introduction-to-dev-containers) page explains it all in more detail.

My `Dockefile` ended up being very simple: a basic Ubuntu image, a list of packages, and a customization so that it runs with a non-root user by default. Something like:
```Dockerfile
FROM ubuntu:jammy

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update
RUN apt-get upgrade -qq -y
RUN apt-get install -qq -y \
    sudo \
    git \
    nano \
    build-essential \
    python3
    # ...
RUN yes | sudo unminimize

# See: https://code.visualstudio.com/remote/advancedcontainers/add-nonroot-user
ARG USERNAME=alnvdl
ARG USER_UID=1000
ARG USER_GID=$USER_UID
RUN groupadd --gid $USER_GID $USERNAME && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME
RUN echo "$USERNAME ALL=(root, postgres) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME && chmod 0440 /etc/sudoers.d/$USERNAME
USER $USERNAME
```

To configure the devcontainer, I then edited `devcontainer.json`:
```jsonc
{
  "build": {
    "dockerfile": "Dockerfile"
  },
  "containerUser": "alnvdl",
  "containerEnv": {
    "SHELL": "/bin/bash",
    "DEBIAN_FRONTEND": "noninteractive",
    "LANG": "en_US.UTF-8",
    "LC_CTYPE": "pt_BR.UTF-8",
    "LC_NUMERIC": "pt_BR.UTF-8",
    "LC_TIME": "pt_BR.UTF-8",
    "LC_COLLATE": "en_US.UTF-8",
    "LC_MONETARY": "pt_BR.UTF-8",
    "LC_MESSAGES": "en_US.UTF-8",
    "LC_PAPER": "pt_BR.UTF-8",
    "LC_NAME": "pt_BR.UTF-8",
    "LC_ADDRESS": "pt_BR.UTF-8",
    "LC_TELEPHONE": "pt_BR.UTF-8",
    "LC_MEASUREMENT": "pt_BR.UTF-8",
    "LC_IDENTIFICATION": "pt_BR.UTF-8",
    "PATH": "${localEnv:PATH}:/home/alnvdl/dev/go/bin:/home/alnvdl/dev/node/bin"
  }
  // ...
}
```

The `PATH` needs to be setup here within `containerEnv`, in addition to being setup in `.bashrc` later. This is needed so VS Code itself (and not only the terminal shell) can find some tools it needs (e.g., `gopls`). See [Installing more things](#installing-more-things) to understand why my `PATH` is built this way.

## Working with more than one repository
By default, a codespace will boot targeting one branch of a specific repository. I wanted to setup my entire VS Code workspace in the cloud, and for that, I needed to allow the codespace to request access to additional repositories:
```jsonc
{
  "customizations": {
    "codespaces": {
      "repositories": {
        "alnvdl/terr": {
          "permissions": "write-all"
        },
        "alnvdl/fstringen": {
          "permissions": "write-all"
        },
        // ...
      }
    }
  }
  // ...
}
```

I could have limited permissions more [as described here](https://docs.github.com/en/codespaces/managing-your-codespaces/managing-repository-access-for-your-codespaces), but these are my personal projects, so I'm fine with having full access.

Another alternative would have been to setup an SSH key that could be used to clone the repositories. But by using the approach above, I get access not just to the code, but also to GitHub issues, pull requests and other stuff via the GitHub extension in VS Code (which seems to come pre-installed by the way).

To add a new repository, the entire codespace needs to be recreated. I usually just delete the old codespace and recreate it from scratch. I think it's very likely that GitHub will eventually make this process better and easier, so maybe these this will change in the future.

## Installing more things
For some of the software I install (e.g., Go, NodeJS), I prefer to download them directly from their official websites to get the latest versions.

To do that, I wrote the `create.sh` script, which downloads, checksums and install these toos locally in my `dev` folder.

I then set that script to run when the codespace is created in `devcontainer.json`:
```jsonc
{
  "onCreateCommand": "/workspaces/workspace/create.sh",
  // ...
}
```

That script also installs a few other tools I use, such as `gopls` and `dlv`, and configures the shell prompt the way I like. It also sets up some locale-related environment variables to my preference.

## Starting background services
By default, containers don't start services.

So to start services like PostgreSQL and Redis, I created a `start.sh` script:
```bash
#!/bin/bash

sudo service postgresql start
sudo -u postgres createuser alnvdl
sudo -u postgres createdb alnvdl

sudo service redis-server start
```

I then set that script to run when the codespace is started in `devcontainer.json`:
```jsonc
{
  "postStartCommand": "/workspaces/workspace/start.sh",
  // ...
}
```

## Setting up the VS Code workspace
Finally, I wrote a VS Code workspace template (`workspace.code-workspace`):
```jsonc
{
  "note": "Do not edit this file, it is auto-generated by create.sh",
  "folders": [
      ${WORKSPACE_REPOS},
      {
          "path": "."
      }
  ]
}
```

And adapted the `create.sh` script to:
1. Clone all the repositories it can find in `devcontainers.json`
2. Fill out the template with the path to all of these repositories

When the codespace starts, you will be prompted to load this workspace file. If you aren't prompted, you can open it manually in VS Code.

I think this workspace setup process may not be needed in the future, but for now it still is. See [Working with more than one repository](#working-with-more-than-one-repository).

## Customizing VS Code with extensions
I don't know if this is strictly needed, or if VS Code setting sync would take care of this. Regardless, you can list the extensions to be installed in VS Code when attached to the codespace in `devcontainer.json`:
```jsonc
{
  "customizations": {
      "vscode": {
          "extensions": [
              "ms-python.python",
              "golang.go",
              // ...
          ]
      }
  }
  // ...
}
```

You can find the IDs for extensions in their marketplace pages (see the [Python extension](https://marketplace.visualstudio.com/items?itemName=ms-python.python) for example).

## Fighting the browser
If you use keyboard shortcuts like `Ctrl+W`, `Ctrl+T` or `Ctrl+N` in VS Code, your browser will intercept them, and that is unavoidable because a website cannot capture these shortcuts in regular mode.

But it's easy to work around these limitations: go full-screen with `F11`. As a bonus, you will gain a few extra pixels :-)

When pressing `F11`, make sure the browser's address bar focused (instead of VS Code having the focus), which will allow you to easily navigate out of the full-screen window with whatever keyboard shortcuts you have defined for that.

If you press `F11` with VS Code focused or enable full-screen via VS Code itself, it will capture all keyboard shorcuts, even the ones used by your desktop environment.

## Persisting shell history
I didn't find a reasonable solution for that. If your codespace gets deleted due to inactivity or because [you needed to add a new repository](#working-with-more-than-one-repository), the entire shell history will be lost.

Looking on the bright side, this may be a way to force people to properly document or persist their most important commands in repositories, instead of keeping them isolated in developers' brains and machines.

## Syncing VS Code settings
To make your VS Code settings persist across different codespaces (or as they get recreated), make sure to [enable the VS Code settings sync](https://code.visualstudio.com/docs/editor/settings-sync).

## Accessing services
To access a service running on a workspace, see [Forwarding ports in your codespace](https://docs.github.com/en/codespaces/developing-in-a-codespace/forwarding-ports-in-your-codespace).

## Bonus: developing GTK applications
GTK ships with a display server called Broadway that can render graphical applications in an HTML canvas and serve it in a page. It works perfectly with a codespace, as can be seen in the screenshot illustrating this post.

It's [very easy to get started with Broadway](https://docs.gtk.org/gtk3/broadway.html), and once it is running, you just need to perform [port forwarding](#accessing-services).
