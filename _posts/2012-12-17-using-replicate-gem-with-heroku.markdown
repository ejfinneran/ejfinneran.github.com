---
layout: post
title: "Using Replicate gem with Heroku"
date: 2012-12-17 17:06
comments: true
categories:
---

### Heroku vs Replicate

#### Update 01/2013:
I've recently found that the `heroku run` command can randomly truncate output so your milage on this may vary, especially with larger sets of data.  GitHub Issue here: <https://github.com/heroku/heroku/issues/674>

While trying to use the [Replicate](https://github.com/rtomayko/replicate) gem with Heroku, I ran into a strange issue.  Namely, Heroku didn't like the bytestream that Ruby's Marshal class was producing:

```
╰─○  heroku run 'replicate -r ./config/environment -d "Post.first"' > file.dump
 !    Heroku client internal error.
 !    Search for help at: https://help.heroku.com
 !    Or report a bug at: https://github.com/heroku/heroku/issues/new

    Error:       invalid byte sequence in UTF-8 (ArgumentError)

```
### Workaround


I did find a workaround though.  The `base64` binary is present on Heroku VMs so you can pipe replicate's output through that to get data that Heroku can handle. However, there are a couple caveats here:

1. Heroku squashes STDERR into STDOUT which means you need to filter out STDERR (via `2>/dev/null`) before sending the data back to the client.

2. You also need to do a little manual work on the data you get back to filter out the lines Heroku addes to the output.

3. Decode the base64 file on your end before passing it your local replicate instance.


### Example

Here is the flow that works for me:

```
heroku run 'replicate -r ./config/environment -d "Post.first" 2>/dev/null | base64' > file.dump
```

Open `file.dump` and remove any of the "…attached to terminal" lines Heroku adds. In the following case, just the first line.

```
Running `replicate -r ./config/environment -d "Post.first" 2>/dev/null | base64` attached to terminal... up, run.7145
BAhbCEkiCVVzZXIGOgZFRmkGewlJIgdpZAY7AFRpBkkiDXVzZXJuYW1lBjsAVEkiCEJvYgY7AFRJ
Ig9jcmVhdGVkX2F0BjsAVFU6IEFjdGl2ZVN1cHBvcnQ6OlRpbWVXaXRoWm9uZVsISXU6CVRpbWUN
```

Decode the base64 file back into a Marshal friendly file and load it into Replicate:

```
╰─○ base64 -i file.dump -D | replicate -r ./config/environment -l
zsh: correct './config/environment' to './config/environments' [nyae]? n
==> loaded 2 total objects:
Post      1
User      1
```

Hope that helps anyone working with Replicate on Heroku.
