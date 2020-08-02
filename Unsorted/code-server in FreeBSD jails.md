# Running `code-server` within FreeBSD jails

## Installation of required packages

`python3` is required for the proper building of `spdlog` by `npm` & `yarn`. `python37` usually gets pulled in when installing `npm` or `yarn` by `pkg`, but the shebang calls for `python3`, and fails to build without either symlinking `python3` to `python37` (`ln -s /usr/local/bin/python37 /usr/local/bin/python3`) or installing the `python3` metaport (which creates a `python3` symlink to whatever the current python 3 version is).

#### Installing with `code14`

```sh
pkg install curl npm python3
```

#### Installing with `code12`

```sh
pkg install curl npm-node12 python3
```

#### Installing with `code14`

```sh
pkg install curl yarn python3
```

#### Installing with `code12`

```sh
pkg install curl yarn-node12 python3
```

## Installation of `code-server`

```sh
curl -fsSL https://code-server.dev/install.sh | sh
```

## Creating an SSL certificate to encrypt traffic



## Assigning a user and isolating `code-server`

#### Creating a `chroot` environment

In addition to setting up a non-root user, we should `chroot` the environment that the `code-server` user uses to prevent access to unnecessary binaries.