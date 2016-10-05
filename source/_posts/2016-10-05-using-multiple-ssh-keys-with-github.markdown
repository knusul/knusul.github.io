---
layout: post
title: "Using multiple ssh-ids with github"
date: 2016-10-05 21:33:37 +0200
comments: true
categories: 
---

Sometimes I need to interact with github using multiple ssh-keys.
This is how I configure git to do it:

Lets create new rsa key
```
ssh-keygen
```

Then configure new key in github
```
  cat ~/.ssh/new_id_rsa.pub
```

When you try to push ssh-key-agent will use your old key. Create/Modify the file ~/.ssh/config
```
Host github.com-new_id_rsa
  HostName github.com
  User git
  IdentityFile ~/.ssh/new_id_rsa
```

Next open `.git/config` in your project and change url accordingly:
```

url = git@github.com-new-id-rsa:YOUR_USERNAME/YOUR_REPOSITORY.github.git
```

Now in your project directory you should be able to git push
