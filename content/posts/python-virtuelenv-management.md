---
title: Python Virtual Environment Mangement
date: 2021-11-16 15:19:22
tags: [python]
---
First, two packages:

```bash
pip3 install virtualenv
pip3 install virtualenvwrapper
```

`virtualenv` makes one can create new virtual environment conveniently, like:

```bash
virtualenv new_env
```

and a new directory named `new_env` will be created under the current directory.

`virtualenvwrapper` is a wrapper of `virtualenv`, which allows one to manage virtual environments.

It supplies some commands:

- `mkvirtualenv`: create a new virtual env
- `rmvirtualenv`: remove a certain virtual env
- `workon`: active or switch virtual env
- `deactivate`: deactivate or exit the virtual env
- `lsvirtualenv`: list all available virtual env

To enjoy the wrapper, put

```bash
export WORKON_HOME='~/.virtualenvs'
source `which virtualenvwrapper.sh`
```

into your favorite shell init file, such as `.zshrc`, `.bashrc`, and then all virtual environments created would be managed under the `~/.virtualenvs` directory.

Sometimes, you should specify which python interpreter is used by

```bash
export VIRTUALENVWRAPPER_PYTHON='/usr/bin/python3'
```

Because the `virtualenvwrapper.sh` will find your `python` rather than `python3`

```bash
# Locate the global Python where virtualenvwrapper is installed.
if [ "${VIRTUALENVWRAPPER_PYTHON:-}" = "" ]
then
    VIRTUALENVWRAPPER_PYTHON="$(command \which python)"
fi
```

But `python` sometimes refers to `python2` or just not exists, such as that on Ubuntu.

Similarly if you want to use the wrapper in python2.
