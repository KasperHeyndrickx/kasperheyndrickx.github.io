---
layout: post
title:  "Configuring multiple git accounts"
date:   2024-05-03 11:20:21 +0100
categories: git github
---

## Git configuration

Git is a set it and forget it setup for most people. Every time I get my hands on a new machine, I always have to go through the same guides. It's not difficult, but since I only do this every year or two, it's not something I tend to remember. Recently I configured git with multiple accounts (private and work), which is a bit more challenging than just blindly following a guide on how to set up an ssh key.

In this post I'll try to explain step by step how to set up git with two accounts. How to commit with different usernames, how to use the right ssh key depending on which repository you're pushing to, and how to sign your commits with different GPG keys.

In this example I assume two github accounts: one private and one work account. On the private repository, I want to make commits with my private email address. On The work repository, I want to make commits with my company email address.

## Git users

I assume you've already set up git with one account. When you run `git config user.name && git config user.email`, it should show your git username and email address. This user data is added to all your commits.

One way to use a different configuration on different repositories is by setting `git config --local user.name "My Name"`. However, the annoying thing is that you have to remember to do this for every new repository. Luckily git introduced [conditional configuration includes](https://git-scm.com/docs/git-config#_includes) in v2.13. In my case, I clone private repositories to `~/Documents/private/repos`, while work repositories go to `~/Documents/work/repos`. To use a different user name and email based on the repository location, I modify `~/.gitconfig` as follows:

```config
[includeIf "gitdir:~/Documents/private/repos"]
  path = .gitconfig-private
[includeIf "gitdir:~/Documents/work/repos"]
  path = .gitconfig-work
```

To set up the actual username and email, I need to create two new files: `~/.gitconfig-work` and `~/.gitconfig-private`. These look like normal git config files. For example, gitconfig-work might look like this:

```config
[user]
        name = my name
        email = name@company.com
```

To test if this works, `cd` to any repository and run `git config user.email`. It should return your private email address on private repositories, and your work email for work repositories. 

## SSH keys

Now that we can `git commit` under different usernames, let's see how we can `git push` using different credentials. To do this we first need to create a new ssh key and add it to our github account. Make sure to give the new key a different name when prompted. I name both keys `~/.ssh/id_ed25519_gh_private` and `~/.ssh/id_ed25519_gh_work`.

```bash
ssh-keygen -t ed25519 -C "name@company.com"
```

Then navigate to your account settings on github and click on ['SSH and GPG keys'](https://github.com/settings/keys). Copy the public key (the file ending in `.pub` in the `~/.ssh/` directory) and add it as a new SSH Key on github. Make sure to log in to the right github account first.

The only thing left to do now is configure git to use the right ssh key on the right repository. Remember the two `.gitconfig` files we created earlier? We'll add an extra section to both files to configure the ssh command. My `~/.gitconfig-work` file now looks like this:

```config
[user]
        name = my name
        email = name@company.com
[core]
        sshCommand = ssh -i ~/.ssh/id_ed25519_gh_work
```

If everything is set up correctly, you should now be able to push and pull from both private and work repositories.

## GPG

The process of initially setting up a GPG key can be confusing, but configuring multiple keys is relatively easy. There's not really any configuration required. If your commit is made with a certain name and email address, then the right key will automatically be used to sign the commit.

As a quick reminder, setting up a new gpg key with github can be done as follows:

```bash
# First generate the key
gpg --full-generate-key

# find the right key id
gpg --list-secret-keys --keyid-format=long

# Find the right id in the output after the algorithm:
# sec    rsa4096/546E539071567BD2 2024-03-06 [SC] [expires: 2028-03-06]
# the key here is '546E539071567BD2'

# export the public key
gpg --armor --export 546E539071567BD2

```
From this output, copy the part beginning with `-----BEGIN PGP PUBLIC KEY BLOCK-----` and ending with `-----END PGP PUBLIC KEY BLOCK-----`. This key should then be added under 'GPG keys' on github.

If you want to add GPG signing to only one account and not for the ohter, then this can also be done by modifying the matching `~/.gitconfig` files. For example:

```config
[commit]
        gpgsign = true / false
```

That's it! Now git is fully set up to work with multiple accounts. 