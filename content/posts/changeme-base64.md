---
title: "'changeme' is valid base64"
date: 2019-12-09T00:10:58+01:00
tags: base64, short-and-sweet
---

[Base64-encoding](https://en.wikipedia.org/wiki/Base64) is ubiquitous in our
modern world. Many programs communicate in base64-encoded messages, since these
have nice properties: They consist of a very limited subset of ASCII characters,
and can thus be displayed as text. E-Mails for example are commonly encoded in
base64 in transfer, which can be seen in the `Content-Transfer-Encoding` header.

Where I volunteer we have a tradition of putting `changeme` as a placeholder for
many template variables indicating that they should be overwritten later. Thus,
if we later still see `changeme` anywhere we know that somewhere somebody must
have forgotten to specify a variable[^1].

Sometimes these template variables should correspond to base64-encoded content,
i.e. the base64-decoding of said variable should produce meaningful content. But
what happens if somebody forgets to define said variable, and it is substituted
with`changeme` instead? I always thought that it would fail to
decode the string since the probability that `changeme` is actually valid base64
encoding must be very low. But lo and behold, this is not the case:

```
% echo changeme | base64 -d
r�[1m
```

Running `hexdump` on the output we can identify the characters:

```
% echo changeme | base64 -d | hexdump -C
00000000  72 16 a7 81 e9 9e      |r.....|
00000006
```

Mostly nonsense, but the second byte is interesting: It is an ASCII control
character [^2], meaning trips up all sorts of programs which expect to read
printable ASCII characters, such as JSON deserializers.

This bug actually occurred when I was migrating some applications between
different Kubernetes clusters: In the old cluster the application ran without
problems, but I forgot to specify some variable in the new cluster, and it lead
to Go complaining about non-ASCII characters in a JSON string.

I guess the moral of the story is: If you use dummy values be sure that they
cannot be misinterpreted.

[^1]: this is not the ideal way to solve this, but we have our historical reasons for choosing this solution.
[^2]: specifically `Ctrl+V`
