---
layout: post
title: "Modifing AWS Gateway output parameters"
date: 2016-02-01 16:05:00 +0000
comments: true
categories:  aws, gateway, rest
---

Maybe you want to hide unnecessery parameters from api or you want to modify the keys. It's straightforawrd with amazon gateway but it took me some time to find how to do it.
Go to AWS Gateway -> Your api -> Integration Response -> Response template and change "passthru" to sth like
```
#set($inputRoot = $input.path('$'))
{
  "listings" : $inputRoot.items
  }

```
This will strip all other keys in a response dict and replace it with one key that's the value of original 'items' key. Whoila!

