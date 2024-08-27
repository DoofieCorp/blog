+++
title = "Seamlessly Switching Between Work and Personal GitHub Accounts with SSH, GPG, and SSH Agent"
description = "Learn how to seamlessly switch between work and personal GitHub accounts using SSH, GPG, and SSH agent configurations. This guide walks you through setting up 1Password for secure key management, configuring Git for different identities, and automating the process to enhance your development workflow."
keywords = [
    "GitHub",
    "SSH",
    "GPG",
    "1Password",
    "Git",
    "SSH agent",
    "Git configuration",
    "multiple GitHub accounts",
    "developer workflow",
    "automation",
    "secure key management"
]
categories = [
 "programming"
]
date = "2024-08-27T00:00:00Z"

+++

<center>

![](/images/github-switching.webp)

</center>

If you're juggling work and personal GitHub accounts, switching between them manually can get frustrating. Luckily, you can set things up so that your environment automatically switches accounts based on the project directory you're working in. This post will show you how to configure your SSH agent with 1Password, use different GPG keys and identities, and make it all work seamlessly.

#### Step 1: Manage SSH Keys with 1Password

1Password’s SSH agent integration can securely manage your SSH keys so you don’t have to deal with passwords or key files every time.

1. Add your SSH keys to 1Password.
2. Enable SSH Agent integration in 1Password so it automatically handles your keys.

Once your keys are stored, associate them with your work and personal GitHub accounts.

#### Step 2: Fetch Public Keys from GitHub

To verify your setup, fetch your public keys from GitHub using these URLs:

- Work account: `https://github.com/work-username.keys`
- Personal account: `https://github.com/personal-username.keys`

#### Step 3: Configure SSH with `IdentitiesFile`

Next, configure your SSH client to use the correct key for each GitHub account. Add the following to your `~/.ssh/config`:

```bash
# Personal GitHub account
Host personal-github
  HostName github.com
  User git
  IdentitiesOnly yes
  IdentitiesFile ~/.ssh/personal_rsa

# Work GitHub account
Host work-github
  HostName github.com
  User git
  IdentitiesOnly yes
  IdentitiesFile ~/.ssh/work_rsa
```

You can test your SSH config with these commands:

```bash
$ ssh work-github
$ ssh personal-github
```

#### Step 4: Load Git Config Based on Directory

To load different Git configurations depending on the directory, add the following to your `~/.gitconfig`:

```ini
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

This will load `~/.gitconfig-work` when you're working in `~/work` and `~/.gitconfig-personal` when in `~/personal`.

#### Step 5: Set Up Specific Git Configs

Now, configure your `~/.gitconfig-work` and `~/.gitconfig-personal` files. These configurations will automatically select the correct SSH profile and Git identity.

**`~/.gitconfig-personal`:**

```ini
[user]
    email = ian@ianduffy.ie
    name = Ian Duffy

[url "git@personalgit:"]
    insteadOf = "git@github.com:"
```

**`~/.gitconfig-work`:**

```ini
[user]
    signingkey = AAAAAAAA
    email = username@work.com
    name = Ian Duffy

[commit]
    gpgsign = true

[url "git@workgit:"]
    insteadOf = "git@github.com:"
```

**How It Works:**  
The `insteadOf` configuration rewrites the GitHub URLs to custom SSH URLs (`git@personalgit:` and `git@workgit:`). These URLs then map to the appropriate SSH `Host` entries in your `~/.ssh/config`, ensuring the correct SSH key is used for each account automatically.

For example, when working in your personal directory, `git@github.com:personal-username/repo.git` becomes `git@personalgit:repo.git`, which uses the SSH key specified in the `Host personal-github` entry.

#### Wrapping It All Up

With this setup, switching between your work and personal GitHub accounts happens automatically based on the project directory. Your email, GPG key, and SSH key will all be correctly configured, saving you from manual switching. Just focus on your code, and let the setup handle the rest!
