+++
title = "Setup multiple git accounts on the same system"
date = "2023-10-20"
draft = false

[taxonomies]
tags = ["git",]

[extra]
lang = "en"
+++

Git is an indispensible part of my workflow. I happen to use git for both my personal and work related projects. However, moving between these accounts can be challenging at times. In order to counter this, we can make use of a feature called [conditional includes](https://git-scm.com/docs/git-config#_conditional_includes).

## Setup ssh keys
To start with let’s set up the ssh keys for each of our accounts. Check for any existing ssh keys in the `.ssh` directory, and remove them if not in use.

- Generate the keys using ssh-keygen

```bash
$ ssh-keygen -t ed25519 -C "you@example.com"
```

This will create two files in your .ssh directory, a private and a public (differentiated with a .pub extension) key. Repeat this step for the second key.

- Add them to your ssh-agent using ssh-add.

```bash
$ ssh-add ~/.ssh/id_ed25519_personal
$ ssh-add ~/.ssh/id_ed25519_work
```

Make sure you add the private key and not the public key. Now add the public keys to your Github/Gitlab accounts.

Navigate to the SSH keys section in your Github/Gitlab accounts and then add your respective SSH keys. Note, copy the contents of the file ending with .pub as this is your public key. Your private key never leaves your computer.

## Creating the ssh config file

Navigate to the .ssh/config file and edit the file similar to below.

```bash
$ cat .ssh/config
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
 
Host work.github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
```

Ensure you are consistent with the `Host` notation, because it will be important later on.

## Setting up the git config

Go to your git config file and add the following

```bash
$ cat .gitconfig 
[user]
        name = you
        email = you@example.com

[includeIf "gitdir:~/work/"]
        path = ~/work/.gitconfig-work
```

And in `~/work/.gitconfig-work` add the following

```bash
[user]
        name = you
        email = you@work.com
```

## Using this setup

Any git directory outside `~/work` will default to your personal user. You can check the user of your work directories using `git config user.email` in any git directory under `~/work`. It should output your work email.

You can clone a directory using:

```bash
git clone git@github.com:personal/repo.git
```

However, for a `~/work` directory, use:

```bash
git clone git@work.github.com:work/repo.git
```

Note, that we use the string we defined as `Host` in the `.ssh/config file` between `@` and `:`. Now, any directory that comes under `~/work` will use your work credentials.

We perform a similar exercise when creating a new repository on the local machine. Initialize your local directory using `git init`. Create the repository on your Github/Gitlab account and then add as a git remote for the local repository.

```bash
git remote add origin git@work.github.com:work/repo.git
```

Now you should be able to easily switch between your different git accounts.