+++
title = "Why do I no longer use Anaconda？"
date = "2022-06-16 21:03:37"
authors = ["茨月"]
+++

This post discusses the pros and cons of Anaconda and why I no longer use it.

<!-- more -->

Today, I finally made up my mind to remove Anaconda, which was taking up more than ten GiB of space. Anaconda played a significant role in my journey of learning and using `Python`, and I’m grateful for its existence.

However, I’ve now abandoned Anaconda and hope that readers studying computer science will also avoid using it. There are three reasons for this:

## Independent Compilers/Dynamic Libraries from the System

The Python versions that come with Linux systems are CPython, compiled by C/C++ compilers. Therefore, if you use the system’s compiler to build a [pybind11](https://pybind11.readthedocs.io/en/stable/) project, the resulting module can be directly recognized by the system’s Python.

Anaconda’s Python, on the other hand, is different: to ensure consistency and ease of use, Anaconda uses pre-built binaries and corresponding versions of dynamic libraries, independent of the system’s compilers and dynamic libraries. As a result, the interoperability between Python in Anaconda’s virtual environment and the system’s compilers & dynamic libraries is poor. This caused significant issues in a project I worked on that used pybind11 (we needed to export C++ interfaces to Python).

It’s undeniable that this "separate ecosystem" approach makes Anaconda work out of the box, but the cost is also substantial.

## PATH Pollution

As mentioned above, Anaconda comes with its own binaries and dynamic libraries, and it includes far more than just Python—tools like OpenSSL, mpic{c|xx}, and many others are also included. These tools make it convenient for Anaconda to maintain its own runtime environment, but when conda adds its `bin` to the PATH, it pollutes the system’s PATH, overriding critical files like the system’s OpenSSL and causing **extremely difficult-to-debug** mysterious bugs.

## Huge Space Consumption

The reason is also mentioned above. To maintain its own environment, Anaconda needs to install various packages and tools. Before I deleted Anaconda, it was taking up 11.4 GiB of space on my system.

## What to Use Instead?

I’m now using [poetry](https://python-poetry.org/). If you need manual multi-version dependency management, consider [pyenv](https://github.com/pyenv/pyenv) with the [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) plugin.
