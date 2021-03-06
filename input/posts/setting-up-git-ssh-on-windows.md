---
Title: "Setting Up Git & SSH on Windows"
Description: "How to set up git to use ssh on Windows using OpenSSH"
Published: 2022-03-12
Image: "img/setting-up-git-ssh-on-windows-cover.jpg"
Tags: ["git", "ssh", "windows", "putty", "github"]
---

I recently had to rebuild my work laptop and had to remember how I had set up [Git](https://git-scm.com),
and I remembered that it was a pain in the arse. But after discovering that Windows 10/11 Pro comes with
an inbuilt [SSH client](<https://en.wikipedia.org/wiki/SSH_(Secure_Shell)>) and SSH-Agent I realised that
it was going to be a lot easier than messing around with PuTTy.

I'm going to assuming you have Git installed already, but if you haven't take a look at using
[chocolatey](https://chocolatey.org/install) to get it installed.

## Getting going with OpenSSH

Check that you've got SSH installed first of all, type `ssh` into your [terminal](https://aka.ms/terminal)
of choice and see if it errors or not. If you don't have it installed you can fire up a powershell window
with Administrator rights and type in the command below.

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client
```

### SSH-Agent

SSH-Agent is a service that manages the SSH keys that we'll create later on.By default it's disabled, so
we'll start off by getting it to run, and making it start up automatically on boot.

```powershell
# (With Admin rights)
Set-Service -Name ssh-agent -StartupType "Automatic"
Start-Service -Name ssh-agent
```

## Creating your SSH Key

If this is your first time using SSH to authenticate with a server then you'll need to create a key.

A key comes in to two parts - public and private. You give the public part to the server, and keep the
private part to yourself. If you're after a more in depth explanation about how it all works then Digital
Ocean have a good article about it
[here](https://www.digitalocean.com/community/tutorials/understanding-the-ssh-encryption-and-connection-process).

We'll create three types of keys:

1. **Ed25519** - Uses elliptic curve cryptography (I have no idea what it means, but it sounds fancy) to produce
   a short key that's quick to process but tough to break and it's the recommended key type to use when you can.
1. **ECDSA** - Is another elliptic curve based key, it's more widely supported than Ed25519 so we'll include it
   as an intermediary between Ed25519 and RSA.
1. **RSA** - The previous default and if you've created keys in the past then it's probably been this type.
   You can pick your key size, but anything below 2048 bits is **not** recommended.

I'm including the RSA key here because not everywhere supports elliptic curve crypto so we'll generate an RSA key
so you can fall back to it if needed. [Azure Devops](https://devops.azure.com) - I'm looking
at you.

```shell
# Ed25519
ssh-keygen -t ed25519 -C "user@host"

#ECDSA
ssh-keygen -t ecdsa -C "user@host"

# RSA with a 4096 bit key
ssh-keygen -t rsa -b 4096 -C "user@host"
```

The text after the `-C` is a comment and is usually used to append an email or some form of identifier.
I tend to use my username and the machine name for this as I generate one key per machine.

When it prompts you to enter a file in which to save the key, it's best to hit return and let it save it
using its default values which is in `.ssh/` in the root of your user directory.

When running the command above it will ask you for a passphrase and where to store the private and public
key files.

If you want to secure your key further with a passphrase you can, but you'll be asked for it every time
it's used. Leaving it without a key means it can be used without any intervention which is useful
in scripts or any automation it's used it.

### Involving Secret Agents

Next we add the keys to ssh-agent so that the service can present them when asked by SSH.
If you've used the default location and file names in the previous step then you just run `ssh-add`
which will pick up the key files in `.ssh/` and list the keys added.

```shell
ssh-add
```

### Always Protect Your Private(s)

You might get the following error when running `ssh-add` or trying to use your ssh keys.

```shell
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions for 'id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "id_rsa": bad permissions
git@github.com: Permission denied (publickey).
```

To fix this you need to open up the advanced permissions tab and disable inheritance and remove all permissions before giving yourself full control of the file. All the details and screenshots are availble at this [StackExchange Answer](https://superuser.com/a/1296046).

You'll need to do this for all the `key files` and the `config` file we'll set up below. And if you know how to do this via powershell then let me know and I'll edit this post to include it.

### Which key to use?

Since we have multiple keys we need to tell OpenSSH which key to use with a
particular host. This is where the [SSH config file](https://www.ssh.com/academy/ssh/config) comes in useful.

Create a file called `config` (no extension) in your `.ssh/` directory, and set it similar to the example below. **Note: The one space
indent before IdentityFile is important**

```shell
Host ssh.dev.azure.com
 IdentityFile ~/.ssh/id_rsa
Host github.com
 IdentityFile ~/.ssh/id_ed25519
Host gitlab.com
 IdentityFile ~/.ssh/id_ed25519
```

## Fixing the Environment

For Git to know which `ssh` binary to use and thus use the `ssh-agent` we configured earlier and provide
our key files, we need to set an environmental variable - `$GIT_SSH` so that it points to ssh.exe which
lives in `C:\Windows\System32\OpenSSH\ssh.exe`.

You might have this already set if you've used PuTTY or TortoiseSVN as this is how they tell git to use
their own SSH binaries.

Using the powershell below will set the environmental variable to the correct value or create it if
it doesn't already exist.

```powershell
[System.Environment]::SetEnvironmentVariable('GIT_SSH','C:\Windows\System32\OpenSSH\ssh.exe')
```

## Git Config

### Your Details

Before doing any work with git you'll need to set it up to use your name and email.

```shell
git config --global user.name 'Rhod'
git config --global user.email rhod@nospamhereplease.com
```

### Renaming the default branch

Historically git has called it's primary branch `master` and in 2020 there was an effort in the tech
community to move away from using loaded phrases that can carry negative meaning for some users. It
looks like the git community has settled on using `main` as the new term for the default and primary
branch. Creating a new repository on [Github](https://github.com/github/renaming) now uses `main` and
git itself has added an option in it's global config to [change the default](https://sfconservancy.org/news/2020/jun/23/gitbranchname/) to something else.

If you're going to do a lot of your work on GitHub then it makes sense to change your
default branch to match GitHub.

```shell
git config --global init.defaultBranch main
```

## Pre-launch

Sometimes when using git from an external GUI such as [Fork](https://git-fork.com/) it will appear to
hang or error out when doing any commands involving SSH. I've found that it's usually on the first connection
to a specific server and happens when SSH is asking if you trust the identity of the server you're connecting
to. Sometimes the GUI doesn't make it apparent that it's waiting for input, or annoyingly - won't let
you input anything.

To avoid this I run a manual ssh connection to any git servers that I'll be interacting. To get the
details we need for this get the SSH details provided by a repo and take the bits before the `:`.
This is the user and the host we need for SSH. For example `git clone git@github.com:rhodcodes/blog.git`
is how I'd clone the repo for this blog.And `git@github.com` is the user and the host we need to
manually connect via SSH.

```shell
ssh git@github.com

The authenticity of host `github.com (140.82.121.4)' cant be established.
RSA Key Fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

If we answer `yes` then SSH will add the host to the `known_hosts` file in `.ssh/` and it will ensure
that we trust any future connections to that host with that particular key fingerprint.

Here are the user and hosts for manually connecting to some of the major Git Hosts

```shell
GitHub:         git@github.com
GitLab:         git@gitlab.com
Azure DevOps:   git@ssh.dev.azure.com
```

If you're having issues connecting run the ssh with verbosity to get it to dump a load of extra info, including which key file it's attempting to use.

```shell
ssh -v git@ssh.dev.azure.com
```

## Conclusion

In this article we've walked through setting up and configuring both git and ssh so they work
together without any future action need on a Windows system.

Thanks for reading.

_Cover photo by [Markus Spiske](https://unsplash.com/@markusspiske?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on
[Unsplash](https://unsplash.com/s/photos/authentication?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)._
