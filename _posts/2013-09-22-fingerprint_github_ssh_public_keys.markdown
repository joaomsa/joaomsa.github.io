---
layout: post
title: "Fingerprint GitHub SSH Public Keys"
date:   2013-09-22 20:12:10
tags: ssh GitHub
---

When introducing GitHub to new employees sometimes the first thing they encounter are authentication problems. 99% of the time these are one of 2 issues, they haven't registered their public keys with a GitHub account or their ssh-agent isn't running with the right private key.

Both problems are fairly straightforward to debug. When encountering this my first instinct is to check if their ssh-agent is running with a private key.

```bash
ssh-add -l
```
Should output something like this, the fingerprint of their public key.

```
2048 e9:bd:b0:97:d5:3c:b5:d5:6f:fe:9d:aa:76:d3:42:0d joaosa@karp (RSA)
```

From here if they've already generated a key add it to their SSH agent or generate a new one:

```bash
if [ ! -f ~/.ssh/id_rsa ] then
    ssh-keygen;
fi; ssh-add ~/.ssh/id_rsa
```

More often though, they may simply have forgotten to register keys with GitHub. 
One of my favorite little known tricks is quickly inspecting a GitHub user's public keys. You can do it by appending `.keys` to the users profile url. You can see mine at [https://github.com/joaomsa.keys](https://github.com/joaomsa.keys).

Problem though is that `ssh-add -l` returns the ssh-keys in a hex fingerprint while GitHub shows them in their raw glory. In order to generate the hex fingerprint of a public key, we can use the built-in option in `ssh-keygen -l`. To quickly fingerprint all public keys a user has registered just run:

```bash
curl -w "\\n" "https://github.com/joaomsa.keys" 2>/dev/null | while read key; do
    ssh-keygen -l -f /dev/fd/0 <<<"$key";
done
```
Now in a more palatable format:

```
2048 d6:0a:99:58:55:fd:bf:0b:d4:21:a2:0d:a2:44:c2:d4 /dev/fd/0 (RSA)
2048 e9:bd:b0:97:d5:3c:b5:d5:6f:fe:9d:aa:76:d3:42:0d /dev/fd/0 (RSA)
```
