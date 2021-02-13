# Maturin

_formerly pyo3-pack_

[![Actions Status](https://github.com/PyO3/maturin/workflows/Run%20tests/badge.svg)](https://github.com/PyO3/maturin/actions)
[![FreeBSD](https://img.shields.io/cirrus/github/PyO3/maturin/master?style=flat-square)](https://cirrus-ci.com/github/PyO3/maturin)
[![Crates.io](https://img.shields.io/crates/v/maturin.svg?style=flat-square)](https://crates.io/crates/maturin)
[![PyPI](https://img.shields.io/pypi/v/maturin.svg?style=flat-square)](https://pypi.org/project/maturin/)
[![Chat on Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg?style=flat-square)](https://gitter.im/PyO3/Lobby)

Build and publish crates with pyo3, rust-cpython and cffi bindings as well as rust binaries as python packages.

This project is meant as a zero configuration replacement for [setuptools-rust](https://github.com/PyO3/setuptools-rust) and [milksnake](https://github.com/getsentry/milksnake). It supports building wheels for python 3.5+ on windows, linux, mac and freebsd, can upload them to [pypi](https://pypi.org/) and has basic pypy support.

## Usage

You can either download binaries from the [latest release](https://github.com/PyO3/maturin/releases/latest) or install it with pip:

```shell
pip install maturin
```

There are three main commands:

 * `maturin publish` builds the crate into python packages and publishes them to pypi.
 * `maturin build` builds the wheels and stores them in a folder (`target/wheels` by default), but doesn't upload them. It's possible to upload those with [twine](https://github.com/pypa/twine).
 * `maturin develop` builds the crate and installs it as a python module directly in the current virtualenv.

`pyo3` and `rust-cpython` bindings are automatically detected, for cffi or binaries you need to pass `-b cffi` or `-b bin`. maturin doesn't need extra configuration files and doesn't clash with an existing setuptools-rust or milksnake configuration. You can even integrate it with testing tools such as [tox](https://tox.readthedocs.io/en/latest/). There are examples for the different bindings in the `test-crates` folder.

The name of the package will be the name of the cargo project, i.e. the name field in the `[package]` section of Cargo.toml. The name of the module, which you are using when importing, will be the `name` value in the `[lib]` section (which defaults to the name of the package). For binaries, it's simply the name of the binary generated by cargo.

## Python packaging basics

Python packages come in two formats: A built form called wheel and source distributions (sdist), both of which are archives. A wheel can be compatible with any python version, interpreter (cpython and pypy, mainly), operating system and hardware architecture (for pure python wheels), can be limited to a specific platform and architecture (e.g. when using ctypes or cffi) or to a specific python interpreter and version on a specific architecture and operating system (e.g. with pyo3 and rust-cpython).

When using `pip install` on a package, pip tries to find a matching wheel and install that. If it doesn't find one, it downloads the source distribution and builds a wheel for the current platform, which requires the right compilers to be installed. Installing a wheel is much faster than installing a source distribution as building wheels is generally slow.

When you publish a package to be installable with `pip install`, you upload it to [pypi](https://pypi.org/), the official package repository. For testing, you can use [test pypi](https://test.pypi.org/) instead, which you can use with `pip install --index-url https://test.pypi.org/simple/`. Note that for publishing for linux, [you need to use the manylinux docker container](#manylinux-and-auditwheel).

## pyo3 and rust-cpython

For pyo3 and rust-cpython, maturin can only build packages for installed python versions. On linux and mac, all python versions in `PATH` are used. If you don't set your own interpreters with `-i`, a heuristic is used to search for python installations. On windows all versions from the python launcher (which is installed by default by the python.org installer) and all conda environments except base are used. You can check which versions are picked up with the `list-python` subcommand.

pyo3 will set the used python interpreter in the environment variable `PYTHON_SYS_EXECUTABLE`, which can be used from custom build scripts.

## Cffi

Cffi wheels are compatible with all python versions including pypy. If `cffi` isn't installed and python is running inside a virtualenv, maturin will install it, otherwise you have to install it yourself (`pip install cffi`).

maturin uses cbindgen to generate a header file, which can be customized by configuring cbindgen through a cbindgen.toml file inside your project root. Aternatively you can use a build script that writes a header file to `$PROJECT_ROOT/target/header.h`.

Based on the header file maturin generates a module which exports an `ffi` and a `lib` object.

<details>
<summary>Example of a custom build script</summary>

```rust
use cbindgen;
use std::env;
use std::path::Path;

fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();

    let bindings = cbindgen::Builder::new()
        .with_no_includes()
        .with_language(cbindgen::Language::C)
        .with_crate(crate_dir)
        .generate()
        .unwrap();
    bindings.write_to_file(Path::new("target").join("header.h"));
}
```

</details>

## Mixed rust/python projects

To create a mixed rust/python project, create a folder with your module name (i.e. `lib.name` in Cargo.toml) next to your Cargo.toml and add your python sources there:

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   └── bar.py
├── Readme.md
└── src
    └── lib.rs
```

maturin will add the native extension as a module in your python folder. When using develop, maturin will copy the native library and for cffi also the glue code to your python folder. You should add those files to your gitignore.

With cffi you can do `from .my_project import lib` and then use `lib.my_native_function`, with pyo3/rust-cpython you can directly `from .my_project import my_native_function`.

Example layout with pyo3 after `maturin develop`:

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   ├── bar.py
│   └── my_project.cpython-36m-x86_64-linux-gnu.so
├── Readme.md
└── src
    └── lib.rs
```

## Python metadata

To specify python dependencies, add a list `requires-dist` in a `[package.metadata.maturin]` section in the Cargo.toml. This list is equivalent to `install_requires` in setuptools:

```toml
[package.metadata.maturin]
requires-dist = ["flask~=1.1.0", "toml==0.10.0"]
```

Pip allows adding so called console scripts, which are shell commands that execute some function in you program. You can add console scripts in a section `[package.metadata.maturin.scripts]`. The keys are the script names while the values are the path to the function in the format `some.module.path:class.function`, where the `class` part is optional. The function is called with no arguments. Example:

```toml
[package.metadata.maturin.scripts]
get_42 = "my_project:DummyClass.get_42"
```

You can also specify [trove classifiers](https://pypi.org/classifiers/) in your Cargo.toml under `package.metadata.maturin.classifiers`:

```toml
[package.metadata.maturin]
classifiers = ["Programming Language :: Python"]
```

Note that `package.metadata.maturin.classifier` is also supported for backwards compatibility.

You can use other fields from the [python core metadata](https://packaging.python.org/specifications/core-metadata/) in the `[package.metadata.maturin]` section, specifically ` maintainer`, `maintainer-email` and `requires-python` (string fields), as well as `requires-external` and `provides-extra` (lists of strings) and `project-url` (dictionary string to string)

## pyproject.toml

maturin supports building through pyproject.toml. To use it, create a `pyproject.toml` next to your `Cargo.toml` with the following content:

```toml
[build-system]
requires = ["maturin"]
build-backend = "maturin"
```

If a `pyproject.toml` with a `[build-system]` entry is present, maturin will build a source distribution (sdist) of your package, unless `--no-sdist` is specified. The source distribution will contain the same files as `cargo package`. To only build a source distribution, pass `--interpreter` without any values.

You can then e.g. install your package with `pip install .`. With `pip install . -v` you can see the output of cargo and maturin.

You can use the options `manylinux`, `skip-auditwheel`, `bindings`, `strip`, `cargo-extra-args` and `rustc-extra-args` under `[tool.maturin]` the same way you would when running maturin directly.  The `bindings` key is required for cffi and bin projects as those can't be automatically detected. Currently, all builds are in release mode (see [this thread](https://discuss.python.org/t/pep-517-debug-vs-release-builds/1924) for details).

For a non-manylinux build with cffi bindings you could use the following:

```toml
[build-system]
requires = ["maturin"]
build-backend = "maturin"

[tool.maturin]
bindings = "cffi"
manylinux = "off"
```

To include arbitrary files in the sdist for use during compilation specify `sdist-include` as an array of globs:

```toml
[tool.maturin]
sdist-include = ["path/**/*"]
```

There's a `cargo sdist` command for only building a source distribution as workaround for [pypa/pip#6041](https://github.com/pypa/pip/issues/6041).

## Manylinux and auditwheel

For portability reasons, native python modules on linux must only dynamically link a set of very few libraries which are installed basically everywhere, hence the name manylinux. The pypa offers special docker images and a tool called [auditwheel](https://github.com/pypa/auditwheel/) to ensure compliance with the [manylinux rules](https://www.python.org/dev/peps/pep-0571/#the-manylinux2010-policy). If you want to publish wheels for linux pypi, **you need to use a manylinux docker image**. The rust compiler since version 1.47 [requires at least glibc 2.11](https://github.com/rust-lang/rust/blob/master/RELEASES.md#version-1470-2020-10-08), so you need to use at least manylinux2010.

maturin contains a reimplementation of a major part of auditwheel automatically checking the generated library. If you want to disable those checks or build for native linux target, use the `--manylinux` flag.

For full manylinux compliance you need to compile in a CentOS 6 docker container. The [konstin2/maturin](https://hub.docker.com/r/konstin2/maturin) image is based on the official manylinux image. You can use it like this:

```
docker run --rm -v $(pwd):/io konstin2/maturin build
```

Note that this image is very basic and only contains python, maturin and stable rust. If you need additional tools, you can run commands inside the manylinux container. See [konstin/complex-manylinux-maturin-docker](https://github.com/konstin/complex-manylinux-maturin-docker) for a small educational example or [nanoporetech/fast-ctc-decode](https://github.com/nanoporetech/fast-ctc-decode/blob/b226ea0f2b2f4f474eff47349703d57d2ea4801b/.github/workflows/publish.yml) for a real world setup.

maturin itself is manylinux compliant when compiled for the musl target. The binaries on the release pages have additional keyring integration (through the `password-storage` feature), which is not manylinux compliant.

## PyPy

maturin can build wheels for pypy with pyo3. Note that pypy [is not compatible with manylinux1](https://github.com/antocuni/pypy-wheels#why-not-manylinux1-wheels) and you can't publish pypy wheel to pypi pypy has been only tested manually and on linux. See [#115](https://github.com/PyO3/maturin/issues/115) for more details.

### Build

```
USAGE:
    maturin build [FLAGS] [OPTIONS]

FLAGS:
    -h, --help
            Prints help information

        --no-sdist
            Don't build a source distribution

        --release
            Pass --release to cargo

        --skip-auditwheel
            Don't check for manylinux compliance

        --strip
            Strip the library for minimum file size

        --universal2
            Control whether to build universal2 wheel for macOS or not. Only applies to macOS targets, do nothing
            otherwise

    -V, --version
            Prints version information


OPTIONS:
    -m, --manifest-path <PATH>
            The path to the Cargo.toml [default: Cargo.toml]

        --target <TRIPLE>
            The --target option for cargo

    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo-extra-args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

            Use as `--cargo-extra-args="--my-arg"`
    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set
        --manylinux <manylinux>
            Control the platform tag on linux. Options are `2010` (for manylinux2010), `2014` (for manylinux2014) and
            `off` (for the native linux tag). Note that manylinux1 is unsupported by the rust compiler. Wheels with the
            native tag will be rejected by pypi, unless they are separately validated by `auditwheel`.

            This option is ignored on all non-linux platforms [default: 2010]  [possible values: 2010, 2014, off]
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
        --rustc-extra-args <rustc-extra-args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`

            Use as `--rustc-extra-args="--my-arg"`
```
### Publish

```
USAGE:
    maturin publish [FLAGS] [OPTIONS]

FLAGS:
        --debug
            Do not pass --release to cargo

    -h, --help
            Prints help information

        --no-sdist
            Don't build a source distribution

        --no-strip
            Do not strip the library for minimum file size

        --skip-auditwheel
            Don't check for manylinux compliance

        --universal2
            Control whether to build universal2 wheel for macOS or not. Only applies to macOS targets, do nothing
            otherwise

    -V, --version
            Prints version information


OPTIONS:
    -m, --manifest-path <PATH>
            The path to the Cargo.toml [default: Cargo.toml]

        --target <TRIPLE>
            The --target option for cargo

    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo-extra-args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

            Use as `--cargo-extra-args="--my-arg"`
    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set
        --manylinux <manylinux>
            Control the platform tag on linux. Options are `2010` (for manylinux2010), `2014` (for manylinux2014) and
            `off` (for the native linux tag). Note that manylinux1 is unsupported by the rust compiler. Wheels with the
            native tag will be rejected by pypi, unless they are separately validated by `auditwheel`.

            This option is ignored on all non-linux platforms [default: 2010]  [possible values: 2010, 2014, off]
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
    -p, --password <password>
            Password for pypi or your custom registry. Note that you can also pass the password through MATURIN_PASSWORD

    -r, --repository-url <registry>
            The url of registry where the wheels are uploaded to [default: https://upload.pypi.org/legacy/]

        --rustc-extra-args <rustc-extra-args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`

            Use as `--rustc-extra-args="--my-arg"`
    -u, --username <username>
            Username for pypi or your custom registry
```

### Develop

```
USAGE:
    maturin develop [FLAGS] [OPTIONS]

FLAGS:
    -h, --help
            Prints help information

        --release
            Pass --release to cargo

        --strip
            Strip the library for minimum file size

    -V, --version
            Prints version information


OPTIONS:
    -b, --binding-crate <binding-crate>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo-extra-args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`

            Use as `--cargo-extra-args="--my-arg"`
    -m, --manifest-path <manifest-path>
            The path to the Cargo.toml [default: Cargo.toml]

        --rustc-extra-args <rustc-extra-args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`

            Use as `--rustc-extra-args="--my-arg"`
```

## Code

The main part is the maturin library, which is completely documented and should be well integrable. The accompanying `main.rs` takes care username and password for the pypi upload and otherwise calls into the library.

The `sysconfig` folder contains the output of `python -m sysconfig` for different python versions and platform, which is helpful during development.

You need to install `cffi` and `virtualenv` (`pip install cffi virtualenv`) to run the tests.

There are two optional hacks that can speed up the tests (over 80s to 17s on my machine). By running `cargo build --release --manifest-path test-crates/cargo-mock/Cargo.toml` you can activate a cargo cache avoiding to rebuild the pyo3 test crates with every python version. Delete `target/test-cache` to clear the cache (e.g. after changing a test crate) or remove `test-crates/cargo-mock/target/release/cargo` to deactivate it. By running the tests with the `faster-tests` feature, binaries are stripped and wheels are only stored and not compressed.

You might want to have look into my by now slightly outdated [blog post](https://blog.schuetze.link/2018/07/21/a-dive-into-packaging-native-python-extensions.html) which explains the intricacies of building native python packages.
