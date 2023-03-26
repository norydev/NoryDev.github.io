---
layout: blogpost
title: You don't need a mysql service in github actions
subhead: Things I whish I knew
categories: code
date: 2023-03-26 10:00:00 -0300
imgclass: cherry-blossom
---

It took me waaaaaayyy to long to understand this. For this reason, I'm writing this short blog post, so that hopefully, you, don't waste your own time on it.

If you are using github actions, and you need a mysql database in your CI script, **you don't need to include a mysql service manually**.

What is this sorcery? Well, if your job runs on `ubuntu-latest`, and at least as of today the 26th of March 2023, `ubuntu-latest` **already includes a mysql service**.

I, of course, didn't know that, and it only took me 🤬 too many tries to figure it out.

![42 tries to setup mysql](/img/posts/many_tries.jpg)

The only thing you have to not forget, is to manually start the mysql service. This could be easily overlooked because other services that you have to manually include (for example postgresql) do start their service automatically.

Here is the script:

```yaml
---
name: CI
on: push
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
    # services:
      # We don't need mysql here because ubuntu-latest
      # includes mysql already.
      # The root user & passwords are root:root
    steps:
      # [...]
      - name: Create DB
        # The mysql service of ubuntu-latest is not
        # started by default, so we first need to
        # start it.
        # Unlike with the postgres service for example,
        # which allows to create a DB when decalring
        # the service,
        # here we need to manually create our mysql
        # database:
        run: |
          sudo /etc/init.d/mysql start
          mysql -e 'CREATE DATABASE my_test_database;' -uroot -proot

        # In this ☝️ script, you probably need
        # some DB setup setps (schema migrations for ex)
        # as well

      # run step requiring the DB
      - name: Run tests
        # [...]
```

Hope it helps. Cheers!
