---
title: "Self-hosted GitHub Actions: The missing bits"
date: 2021-02-15T14:41:29-05:00
draft: false

slug: github-actions-hosted # Will take the filename as the slug.

description: "The missing bits that aren't well documented when using self-hosted Github Actions runners"
tags: ["github", "github-actions", "self-hosted"]
---

Having your own self-hosted GitHub Actions runner can be advantageous for quite a few reasons. Perhaps you need longer running jobs (a separate discussion on why you shouldn't need this should be happening however), or perhaps you have something hosted internally that you can't quite open up to GitHub Actions externally that you need to access during your build. Whatever the reason, GitHub at least provides you the opportunity to host your own for whatever the situation. While the documentation is good, it's not exactly great. That is, if you want to do anything other than use their built-in scripts and assumptions. If you truly need to customize how you are hosting things, then these tips may help you out a bit.

## The directory assumptions
### HOME
While you can technically have any user run the runner application, there are a few actions and tools that make the assumption that your home directory is `/home/runner`. If you choose to use another user other than `runner` simply linking your home directory to `/home/runner` should make sure all is well.

### AGENT_TOOLSDIRECTORY
This assumption is two fold. First off, that you have exported an environment variable with the name `AGENT_TOOLSDIRECTORY`, and that it always has a value of `/opt/hostedtoolcache`. Sadly, this is hardcoded in quite a few places in various actions as a left over from GitHub's own runners. This particularly is assumed in most of the language runtime actions, like [actions/setup-python](https://github.com/actions/setup-python). You also have to make sure that this directory exists, and that whichever user you chose to run the runner has permissions to it.

### A work directory
To enable being able to clear out things between action runs, we'll want a separate work directory. One of the features listed in the official documentation is "Don't need to have a clean instance for every job execution" so you don't need to clear this out, but should you wish to, you'll want to set this up separately from say, your home directory.

## Dependencies
The runner package has a `./bin/installdependencies.sh`, which should install most dependencies required for the runner. This doesn't necessarily install packages that will be required for most actions. So, we'll want to install a few more. Assuming you are using debian/ubuntu base, you'll want:

- git
- libyaml
- buildessential

We'll also want a few more things to script out how we start up the runner
- curl
- jq

Additionally, you'll absolutely want to make sure that the time on whatever system you choose is synced to a timeserver. Any issues with time will prevent the runner from registering with GitHub.

## Runner startup process
### Registration
To properly register a runner with GitHub, you have to do some token exchange. This starts with having a [GitHub personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token). Once you have that, you next have to use it to get a specific registration token for your runner. So, let's register a runner for our whole organization.

```sh
GITHUB_TOKEN=MY_TOKEN_HERE
OWNER=MY_GITHUB_ORG
# Get runner authentication token
REGISTRATION_TOKEN=$(curl -s -XPOST \
    -H "authorization: token ${GITHUB_TOKEN}" \
    "https://api.github.com/orgs/${OWNER}/actions/runners/registration-token" | jq -r .token)
```

### Configuration
Next, you can configure and start up your runner by providing the proper parameters, including your token. You'll want to call the `config.sh` script included in the runner package.

```sh
WORKDIR=/my/workdir/path
mkdir -p "${WORKDIR}"
./config.sh \
      --url "https://github.com/${OWNER}" \
      --token "${REGISTRATION_TOKEN}" \
      --name "MY_RUNNER_NAME" \
      --work "${WORKDIR}" \
      --replace
```

### Starting

By providing a particular parameter to the included `./run.sh` you can have the runner exit once an single action has run on your runner. Using this mechanism along with system level control that forces the runner to always be running, we can clean up after ourselves after exit, and unregister our runner. We can make this a function to trap on in our script.

```sh
cleanup() {
  # Deregister the runner from github
  REGISTRATION_TOKEN=$(curl -s -XPOST \
      -H "authorization: token ${GITHUB_TOKEN}" \
      "https://api.github.com/orgs/${OWNER}/actions/runners/registration-token" | jq -r .token)
  ./config.sh remove --token "${REGISTRATION_TOKEN}"

  # Remove our runner work dir to clean up after ourselves
  rm -rf ${WORKDIR}
}

trap cleanup EXIT
./run.sh --once
```

## Conclusion
If you want to run Docker based actions, you will also have docker installed, in the path, and you'll want to add some parts to clean up docker volumes and images. If you decide to try to create a docker container to start up a runner, keep in mind that you'll have to deal with docker-in-docker issues if you use docker-based actions. You can try to keep a runner up once it exits by creating a `systemd` service with `Restart=always`. Want to run a bunch on a large system, then make sure that you give them different names. With the `--replace` parameter, if you use the same name, any new runner that registers itself will just do what you told it to do, replace the last one with the same name.

Hopefully this helps you with your own self-hosted runners if you want to control the process a bit more than what is included in the main documentation.
