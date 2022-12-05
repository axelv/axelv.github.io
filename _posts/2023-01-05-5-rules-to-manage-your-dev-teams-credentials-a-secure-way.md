---
layout: post
title: "How to manage credentials and secrets on MacOS"
---

# How to manage credentials and secrets on MacOS

**Note:** Credentials can be anything that authenticates you or grants access to a sensitive system. Ex: API keys, username-password combinations, private-public keys, certificates

## Rule 0 - **Don't** hard code/configure secrets and credentials into source code or any files

This should be obvious. **Do** use environment variables to inject secrets into your code! Code will be copy-pasted and shared in the future, and so will the hardcoded secrets in your code.

## Rule 1 - **Don't** store keys in environment files (.env, ~/.zshrc)

Keys stored in _.env_ or _.zshrc_ risk to be included in some form of version control or backup system. And git repositories will eventually be shared with freelancers, clients or other stakeholders. The potential security breach is huge.

**Do store your keys in KeyChain**
MacOS has a specific application to store your passwords and other secrets. You can easily include them in your dev environment by implementing the following two steps:

1.  Put the file `keychain-environment-variables.sh` somewhere in your home directory or subdirectory and call `source ~/keychain-environment-variables.sh` in your `.zshrc` file. This will create two helper functions that will allow you to easily import keys from your KeyChain into environment variables

2.  Use a `.env` file for each project that read your secrets into environment variables for that project. Avoid putting secrets in environment variables that are exported in `~/.zshrc`. Try to isolate them for the project that you need them for. You can copy secrets in to your environment variables by including the following statement in your `.env` .

```
export YOUR_SECRET_ENV_VAR = $(keychain-environment-variable SECRET_IN_KEYCHAIN)
```

<iframe 
    width="100%"
    height="350"    
    src="https://gist.github.com/axelv/73b6f2c748f9677d53b225fb1104f757.pibb"
 />
    
 
[Open the Gists here if the iframe bellow doesn't render correctly.](https://gist.github.com/axelv/73b6f2c748f9677d53b225fb1104f757)
    
## Rule 2 - **Don't** copy or mount secrets in Docker containers
Be aware that containers will live their own live and you can't predict where they will end up. You don't know who will need access to your container registry in 6 months. You don't know to who you will share your container with. And over time, chances are high you even forgot that there are secrets in your container image. Depending on your needs, there are two approaches to include secrets in your docker deployments:

### 1. My container needs access to a sensitive system at _runtime_.

**Do pass your secrets as environment variables to your container.**
`docker run` allows you to [set both environment variables and entire environment files](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables--e---env---env-file) using arguments `-e` or `--env-file`. When using this option, keep in mind [[#Rule 1 - Don't store keys in environment files env zshrc]] when working with environemnt files.

### 2. My container needs access to a sensitive system at _buildtime_.

**Do mount your secrets using `--mount` in your Dockerfile**
Using buildkit you can provide information and files during [buildtime](https://docs.docker.com/develop/develop-images/build_enhancements/) only.
There is a special option to mount _secrets_: [New Docker Build secret information](https://docs.docker.com/develop/develop-images/build_enhancements/#new-docker-build-secret-information)

And also an option to mount your _ssh-agent_: [Using SSH to access private data in builds](https://docs.docker.com/develop/develop-images/build_enhancements/#new-docker-build-secret-information) Make sure you also set you add your ssh host to the `known_hosts` file. For example to add GitHub.com:
`RUN ssh-keyscan github.com > /etc/ssh/ssh_known_hosts`

**Note:** Be aware that your KeyChain manages your _ssh-agent_ and you don't need to run it yourself. More info here [^2]

Sources:

- [^1] [Superchared Docker Build with BuildKit — DockerCon EU 2018](https://www.youtube.com/watch?v=kkpQ_UZn2uo&t=1084s)
- [^2] [Understanding SSH-keys and using KeyChaing to managed passphrase on MacOS](https://rderik.com/blog/understanding-ssh-keys-and-using-keychain-to-manage-passphrase-on-macos/)
