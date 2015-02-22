# Homu

[![Hommando](https://i.imgur.com/4RfmXP9.png)](https://wiki.puella-magi.net/Homura_Akemi)

Homu is a continuous integration service bot that works with GitHub and
[Buildbot](http://buildbot.net/), or [Travis CI](https://travis-ci.org/). It is
largely inspired by [bors](https://github.com/graydon/bors).

## Why is it needed?

Let's consider Travis CI as an example. If you send a pull request to a
repository, Travis CI instantly shows you the test result, which is great.
However, after several other pull requests are merged into the `master` branch,
your pull request can *still* break things after being merged into `master`. The
traditional continuous integration solutions don't protect you from this.

In fact, that's why they provide the build status badges. If anything pushed to
`master` is free from breakage, the badges are **not** necessary. The badges
themselves prove that there can still be some breakages, even when continuous
integration services are used.

To solve this problem, the test procedure should be executed *just before* the
merge, not after a pull request is received. Homu automates this process. It
listens to the pull request comments, waiting for an approval from one of the
reviewers. When the pull request is approved, Homu tests it using your favorite
continuous integration service, and only if it passes the tests, it is merged
into `master`.

Note that Homu is **not** a replacement of Travis CI, or Buildbot. It works on
top of them. Homu itself doesn't have the ability to test pull requests.

## Influences of bors

Homu is basically a rewrite of bors, which shares the same concept that tests
should be done just before the merge. However, there are also some differences:

1. Stateful: Unlike bors, which intends to be stateless, Homu is stateful.
   It means that Homu does not need to retrieve all the information again and
   again from GitHub at every run. This is essential because of the GitHub's
   rate limiting. Once it downloads the initial state, the following changes
   are delivered with the [Webhooks](https://developer.github.com/webhooks/)
   API.
2. Pushing over polling: Homu prefers pushing wherever possible. The pull
   requests from GitHub are retrieved using Webhooks, as stated above. The
   test results from Buildbot are pushed back to Homu with the
   [HttpStatusPush](http://docs.buildbot.net/current/manual/cfg-statustargets.html#httpstatuspush)
   feature. This approach improves the overall performance and the response
   time, because the bot is informed about the status changes immediately.

And also, Homu has more features, such as `rollup`, `try`, and Travis CI
support.

## Usage

### How to install

```sh
sudo apt-get install python3-venv

pyvenv .venv
.venv/bin/pip install -e .
```

### How to configure

1. Copy `cfg.sample.toml` to `cfg.toml`, and edit it accordingly.

2. Add a Webhook to your repository:

 - Payload URL: `http://HOST:PORT/github`
 - Content type: `application/json`
 - Secret: The same as `repo.NAME.github.secret` in cfg.toml

3. Add a Webhook to your continuous integration service:

 - Buildbot

   Insert the following code to the `master.cfg` file:

   ```python
  from buildbot.status.status_push import HttpStatusPush

  c['status'].append(HttpStatusPush(
      serverUrl='http://HOST:PORT/buildbot',
      extra_post_params={'secret': 'repo.NAME.buildbot.secret in cfg.toml'},
  ))
   ```

 - Travis CI

   Insert the following code to the `.travis.yml` file:

   ```yaml
   notifications:
       webhooks: http://HOST:PORT/travis

   branches:
       only:
           - auto
           - try
   ```

### How to run

```sh
.venv/bin/homu
```
