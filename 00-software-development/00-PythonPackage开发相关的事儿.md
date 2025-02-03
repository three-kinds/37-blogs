# PythonPackage开发相关的事儿

## 1. 如何让我们的代码变成一个Package

* https://packaging.python.org/en/latest/tutorials/packaging-projects/

## 2. 如何通过pip使用我们的Package

### 上传到pypi.org上所有人都能使用

* https://packaging.python.org/en/latest/tutorials/packaging-projects/#uploading-the-distribution-archives

### 使用nexus搭建pypi私服，内部使用 

* https://help.sonatype.com/en/pypi-repositories.html

## 3. 如何让我们的Package更有范儿

* 下面有一些库，是大佬们通常用到的，我们可以按需使用

### tox - 自动化测试工具

* https://github.com/tox-dev/tox
* 配置通常在`tox.ini`文件
* 通过不同Python版本的测试，可以让我们的Package使用范围更广

### coverage - 代码覆盖率工具

* https://github.com/nedbat/coveragepy
* 配置通常在`pyproject.toml`文件中，或者`.coveragerc`文件中
* 它可以统计代码的覆盖率，100%覆盖非常爽

### pytest - 单元测试工具

* https://github.com/pytest-dev/pytest
* 配置通常在`pyproject.toml`文件中，或者`.pytest.ini`文件中
* 我更喜欢用Python内置的`unittest`，因为可以敲更多的代码

### flake8 - 代码风格检查工具

* https://github.com/PyCQA/flake8
* 配置通常在`pyproject.toml`文件中，或者`.flake8`文件中
* 但它做不了类型检查，需要配合mypy一起使用

### mypy - 静态类型检查工具

* https://github.com/python/mypy
* 配置通常在`pyproject.toml`文件中，或者`.mypy.ini`文件中
* 它也有无能为力的时候，比如元编程，这时候就需要`stubs/pyi`文件了

### pyright - 静态类型检查工具

* https://github.com/microsoft/pyright
* 配置通常在`pyproject.toml`文件中
* microsoft出的，可以和mypy一起使用

### isort - 代码格式化工具

* https://github.com/PyCQA/isort
* 配置通常在`pyproject.toml`文件中
* 规范import的顺序

### black - 代码格式化工具

* https://github.com/psf/black
* 配置通常在`pyproject.toml`文件中
* 自动格式化代码，让代码风格一致

### ruff - 目标是替换flak8 + black + isort + 等等的工具

* https://github.com/astral-sh/ruff
* 配置通常在`pyproject.toml`文件中
* Rust写的，速度快
