# bash_unit_testing_using_bats

Some documentation about how to write unit tests scenarios for Bash scripts, using Bats.

>[https://github.com/bats-core](https://github.com/bats-core)

## Installation

### Using git submodules

Just run the following commands in your repository root directory to create a new `~/test` folder with two subfolders `~/test/bats` and `~/test/test_helper`.

```bash
git submodule init
git submodule add https://github.com/bats-core/bats-core.git test/bats
git submodule add https://github.com/bats-core/bats-support.git test/test_helper/bats-support
git submodule add https://github.com/bats-core/bats-assert.git test/test_helper/bats-assert
git submodule add https://github.com/bats-core/bats-file test/test_helper/bats-file
git submodule https://github.com/lox/bats-mock test/test_helper/bats-mock
```

*The two last submodules here above (`bats-file` and `bats-mock` f.i.) are not mandatory but enhanced the feature of bats like allowing to check the existence of files/folders and to use mock feature.*

### Using Docker

Bats can be used in a Dockerized way.

```dockerfile
FROM bats/bats:latest

mkdir -p /opt/bats-test-helpers
git clone https://github.com/ztombol/bats-support test/test_helper/bats-support
git clone https://github.com/ztombol/bats-assert test/test_helper/bats-assert
git clone https://github.com/bats-core/bats-file test/test_helper/bats-file
git clone https://github.com/lox/bats-mock test/test_helper/bats-mock

WORKDIR /code/
```

Then buil the image:

```bash
docker build . -t my/bats:latest
```

## Create a test scenario

Imagine the following, simplified, tree structure:

```text
.
├── src
│   └── helper.sh
└── test
    ├── test.bats
```

The file `~/src/helper.sh` contains your code i.e. a function you want to test.

You test scenario should be copied in the `~/test` folder. We'll call this file `test.bats`.

Below the content of the `~/src/helper.sh`. Very simple function to check the existence of a file on the filesystem. This very straight-forward function will return `0` if the file exists and `1` otherwise. The filename has to be passed as a parameter to the function.

```bash
#!/usr/bin/env bash

function assert::fileExists() {
    [[ -f "$1" ]]
}
```

Below the content of the `~/test/test.bats`.

```bash
setup() {
    # bats-assert was installed using `git submodule add https://github.com/bats-core/bats-assert.git test/test_helper/bats-assert`
    load 'test_helper/bats-assert/load'

    # These two lines to add the `src` folder in the PATH so we don't need
    # to repeat everytime `../src/` in our `@test` function below.
    DIR="$(cd "$(dirname "$BATS_TEST_FILENAME")" >/dev/null 2>&1 && pwd)"
    PATH="$DIR/../src:$PATH"
}

@test "Assert file exists on an existing file" {
    source helper.sh
    run assert::fileExists "src/helper.sh"
    assert_success
}

@test "Assert file exists on an not existing file" {
    source helper.sh
    run assert::fileExists "unexisting_file.sh"
    assert_failure
}
```

## Run the tests

The binary to use is `./test/bats/bin/bats` and we need to specify where our tests are stored (in our case, in the `test` folder). To run all `.bats` file present in the folder:

```bash
clear ; ./test/bats/bin/bats test
```

We can, of course, run only a specific file; like our `test.bats`:

```bash
clear ; ./test/bats/bin/bats test/test.bats
```

By default, only `.bats` file under a given directory are fired. We can go recursively using the `--recursive` flag.

We can, too, use the `--filter` flag where we can use regex. The filter action will NOT be done on the filenames but on their description.  To fire f.i. all tests having the words `not existing` in their name:

```bash
clear ; ./test/bats/bin/bats --filter "not existing" test
```

### Run using the Docker image

If you're using Docker, here is the command to execute all tests:

```bash
docker run -it -v ${PWD}:/code my/bats /code/test
```

or a specific one:

```bash
docker run -it -v ${PWD}:/code my/bats /code/test/test.bats
```

## Mocking

[https://github.com/dodie/testing-in-bash/tree/master/example-bats#mocking](https://github.com/dodie/testing-in-bash/tree/master/example-bats#mocking)

Mocking is possible with all [three common techniques](https://github.com/dodie/testing-in-bash/tree/master/mocking):

* alias
* function export
* PATH override


## Additional ressources:

* [Bats Official documenation](https://bats-core.readthedocs.io/en/stable/)
* [https://github.com/bats-core](https://github.com/bats-core)
* [https://github.com/dodie/testing-in-bash](https://github.com/dodie/testing-in-bash)
* [https://marck-oemar.medium.com/unusual-unit-testing-part-1-bash-scripts-with-bats-55ac78e61491](https://marck-oemar.medium.com/unusual-unit-testing-part-1-bash-scripts-with-bats-55ac78e61491) / [https://github.com/marck-oemar/unittesting](https://github.com/marck-oemar/unittesting)
