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
```

*The last submodule here above (`bats-file`) is not mandatory but enhanced the feature of bats like allowing to check the existence of files/folders.*

### Using Docker

Bats can be used in a Dockerized way.

```dockerfile
FROM bats/bats:latest

mkdir -p /opt/bats-test-helpers
git clone https://github.com/ztombol/bats-support test/test_helper/bats-support
git clone https://github.com/ztombol/bats-assert test/test_helper/bats-assert
git clone https://github.com/bats-core/bats-file test/test_helper/bats-file

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

### Somes examples

#### assert_equal

Simply verify that both values are equals. Here, we'll call a function that will return the lenght of an array and verify it's the expected value.

```bats
@test "array::length - Calculate the length of an array" {
    arr=("one" "two" "three" "four" "five")

    assert_equal $(array::length arr) 5
}
```

```bash
function array::length() {
    local -n array_length=$1
    echo ${#array_length[@]}
}
```

#### assert_failure

`assert::binaryExists` will exit `1` if the binary can't be retrieved. An error message like `The binary can't be found` will be echoed on the console. 

```bats
@test "Assert binary didn't exists" {
    # Simulate which and return an error meaning "No, that binary didn't exists on the host"
    which() {
        exit 1
    }

    run assert::binaryExists "Inexisting_Binary" "Ouch, no, no, that binary didn't exists on the system"
    assert_output --partial "Ouch, no, no, that binary didn't exists on the system"
    assert_failure
}
```

```bash
function assert::binaryExists() {
    local binary="${1}"
    local msg="${2:-${FUNCNAME[0]} - File \"$binary\" did not exists}"

    [[ -z "$(which "$binary" || true)" ]] && echo "$msg" && exit 1

    return 0
}
```
#### assert_output

`assert_output` has two nice options: `--partial` and `--regexp`.

Using `--partial` will allow f.i. the check if the output contains some raw text like, for a help screen, a given sentence.

```bats
@test "Show the help screen" {
    source 'src/decktape/decktape.sh'
    arrArguments=("--help")
    run decktape::__showHelp ${arrArguments[@]}
    assert_output --partial "Convert a revealJs slideshow to a PDF document"
}
```

Using `--regexp` will allow to use a regular expression:

```bats
@test "Show the help screen" {
    source 'src/decktape/decktape.sh'
    arrArguments=("--input InvalidFile")
    run decktape::__process ${arrArguments[@]}
    assert_output --regexp "ERROR - The input file .* doesn't exists."
}
```

#### assert_success

`assert::binaryExists` will return `0` when the binary can be retrieved. The function will run in silent (no output).

```bats
@test "Assert binary exists" {
    run assert::binaryExists "clear" # clear is a native Linux command
    assert_output ""                 # No output when success
    assert_success
}
```

```bash
function assert::binaryExists() {
    local binary="${1}"
    local msg="${2:-${FUNCNAME[0]} - File \"$binary\" did not exists}"

    [[ -z "$(which "$binary" || true)" ]] && echo "$msg" && exit 1

    return 0
}
```


#### Check for ANSI colors

Imagine the following code:

```bash
__RED=31
function console::printRed() {
    for line in "$@"; do
        printf "\e[1;${__REd}m%s\e[0m\n" "$line"
    done
}
```

We wish to check that the line will be echoed in red.

```bats
@test "console::printRed - The sentence should be displayed" {
    run console::printRed "This line should be echoed in Red"
    assert_output "[1;31mThis line should be echoed in Red[0m"
    assert_success
}
```

#### Check for multi-lines output

Imagine the following code:

```bash
function console::banner() {
    printf "%s\n" "============================================"
    printf "= %-40s =\n" "$@"
    printf "%s\n" "============================================"
}
```

This will write three lines on the console, like f.i. 

```text
# ============================================
# = Step 1 - Initialization                  =
# ============================================
```

To check for multi-lines, use the `$lines` array like this:

```bats
@test "console::banner - The sentence should be displayed" {
    run console::banner "Step 1 - Initialization"
    assert_equal "${lines[0]}" "============================================"
    assert_equal "${lines[1]}" "= Step 1 - Initialization                  ="
    assert_equal "${lines[2]}" "============================================"
    assert_success
}
```

#### Check against a file

Imagine a function that will parse a file and f.i. remove some paragraphs. We need to check if the content is correct, once the function has been fired.

For this, imagine a `removeTopOfFileComments` function. The function will parse the file and remove the HTML comments (`<!-- ... -->`) present at the top of the file.

For the test, we'll create a file with three empty lines, then a HTML comment block, then two empty lines, then the HTML code. So, by removing the HTML comment, we'll have five empty lines followed by the HTML block so, we need to check our file contains six lines.

The tip used is:

* `cat --show-ends --show-tabs "$tempfile"` i.e. get the content of the file but with `$` where we've a linefeed and, here, also `^I` for tabs.
* then we'll pipe the result with `tr "\n" "#"` so, instead of getting six lines, we'll get only one by replacing linefeed by `#`.

Now, bingo, since we've a variable with only one line (in our example: `$#$#$#$#$#<html><body/></html>$#`), we can compare with our expectation:

```bats
@test "html::removeTopOfFileComments - remove HTML comments - with empty lines" {
    tempfile="$(mktemp)"
    
    # Here, we'll have extra, empty, lines. They should be removed too
    echo '' >$tempfile
    echo '' >>$tempfile
    echo '' >>$tempfile
    echo '<!--  ' >>$tempfile # We also add extra spaces before the start tag
    echo '   Lorem ipsum dolor sit amet, consectetur adipiscing elit.' >>$tempfile
    echo '   Morbi interdum elit a nisi facilisis pulvinar.' >>$tempfile
    echo '   Vestibulum fermentum consequat suscipit. Vestibulum id sapien metus.' >>$tempfile
    echo '-->     ' >>$tempfile # We also add extra spaces after the end tag
    echo '' >>$tempfile
    echo '' >>$tempfile
    echo '<html><body/></html>' >>$tempfile

    run html::removeTopOfFileComments "$tempfile"

    # Get now the content of the file
    #   We expect three empty lines (the three first)
    #       The HTML comment has been remove
    #   Then there are two more empty line (so we'll five empty lines)
    #   And we'll have our "<html><body/></html>" block.
    #
    #   cat --show-ends --show-tabs will show the dollar sign (end-of-line) and f.i. ^I for tabulations
    #   tr "\n" "#" will then convert the linefeed character to a diese so, in fact, fileContent will
    #   be a string like `$#$#$#$#$#<html><body/></html>$#`
    fileContent="$(cat --show-ends --show-tabs "$tempfile" | tr "\n" "#")"

    # Once we've our string, compare the fileContent with our expectation
    assert_equal "$fileContent" "\$#\$#\$#\$#\$#<html><body/></html>\$#"
}
```

#### Check against a file using a regex

A second scenario can be: you have a `write` function (think to a logfile) and you want to check the presence of a given line in the file.

The example below relies on [bats-file](https://github.com/ztombol/bats-file) and his `assert_file_contains` method.  That method ask for a filename and a regex pattern.

```bash
setup() {
    load 'test_helper/bats-support/load'
    load 'test_helper/bats-assert/load'
    load 'test_helper/bats-file/load'

    #! "grep" without the "-P" argument seems to not support repetition like "\d{4}"
    #
    # a date like `2022-04-07`
    regexDate="[0-9][0-9][0-9][0-9]\-[0-9][0-9]\-[0-9][0-9]"
    # a time like `17:41:22`
    regexTime="[0-9][0-9]\:[0-9][0-9]\:[0-9][0-9]"
    # a timezone difference like "0200"
    regexUTC="[0-9]*" #! Should be [0-9][0-9][0-9][0-9] but didn't work???

    # The final pattern so we can match f.i. `[2022-04-07T18:00:20+0200] `
    __DATE_PATTERN="\[${regexDate}\T${regexTime}\+${regexUTC}.*\]\s"

    return 0
}


@test "log::write - Write a line in the log" {
    local sentence=""
    sentence="This is my important message"
    run write "${sentence}"

    assert_file_exist "/tmp/bats_log.tmp"

    echo "${__DATE_PATTERN}${sentence}" >/tmp/regex.tmp
    assert_file_contains "/tmp/bats_log.tmp" "${__DATE_PATTERN}${sentence}"
    assert_success
}
```

#### Check that a value is NOT in a file

Another use of the `assert_failure` can be to start a command like a grep and expect to get an error:

```bash
run grep "REGEX_SOMETHING_THAT_SHOULD_BE_MISSING" "/tmp/test.log"
assert_failure 1
```

## Some special functions

### setup

The setup function is called before running a test. For each `@test` function present in the scenario, the `setup()` function will be called.

In the following example, since there are two test functions, `setup()` will be called twice.

```bats
setup() {
    load 'test_helper/bats-support/load'
    load 'test_helper/bats-assert/load'

    ENV_ROOT_DIR=""

    source 'src/env.sh'
}

@test "env::assertFileExists - Assert .env file exists - The path isn't initialized" {
    run env::assertFileExists
    assert_failure
}

@test "env::assertFileExists - Assert .env file exists - The file exists" {
    ENV_ROOT_DIR="/tmp"
    ENV_FILENAME=".env.bats.testing"
    touch ${ENV_ROOT_DIR}/${ENV_FILENAME}
    run env::assertFileExists
    assert_success
}
```

### teardown

Just like `setup`, the `teardown` function will be called for each tests but once the test has been fired. This is the good place for, f.i., removing some files created during the execution of a test.

```bats
teardown() {
    rm -f /tmp/bats
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

We can override a function during a test. Consider the following use case: we've a function that will return `0` when a give Docker image is present on the host. The function will return `1` and echo an error on the console if the image isn't retrieved.

```bash
function assert::dockerImageExists() {
    local image="${1}"
    local msg="${2:-The Docker image \"$image\" did not exists}"

    # When the image exists, "docker images -q" will return his ID (f.i. `5eed474112e9`), an empty string otherwise
    [[ "$(docker images -q "$image" 2>/dev/null)" == "" ]] && echo "$msg" && exit 1

    return 0
}
```

So, we need to *override* the docker answer. When the image is supposed to be there, we just need to return a non-empty string, anything but not an empty string. Let's return a fake ID to really simulate the answer of `docker images -q`.


```bats
@test "Assert docker image exists" {
    # Mock - we'll create a very simple docker override and return a fake ID
    # This will simulate the `docker images -q "AN_IMAGE_NAME"` which return
    # the ID of the image when found
    docker() {
        echo "feb5d9fea6a5"
    }

    source assert.sh
    run assert::dockerImageExists "A-great-Docker-Image"
    assert_output "" # No output when success
    assert_success
}
```

And return an empty string to simulate an inexisting image.

```bats
@test "Assert docker image didn't exists" {
    # Mock - we'll create a very simple docker override and return a fake ID
    # This will simulate the `docker images -q "AN_IMAGE_NAME"` which return
    # the ID of the image when found; here return an empty string to simulate
    # an inexisting image
    docker() {
        echo ""
    }

    source assert.sh
    run assert::dockerImageExists "Fake/image" "Bad choice, that image didn't exists"
    assert_output --partial "Bad choice, that image didn't exists"
    assert_failure
}
```

Run the test

```bash
clear ; ./test/bats/bin/bats test/assert.bats --filter "Assert docker"
```

And we'll get this:

```text
✓ Assert docker image exists
✓ Assert docker image didn't exists

2 tests, 0 failures
```

You can find another example of how to mock a function here:

* [assert_failure](#assert_failure)

## Special cases

### The test must always fail

>[https://github.com/ztombol/bats-support#fail](https://github.com/ztombol/bats-support#fail)

If a test should always fail (for instance because it's not yet correctly coded); use the `fail´ verb:

```bats
@test 'fail()' {
  fail 'this test always fails'
}
```

## Additional ressources:

* [Bats Official documenation](https://bats-core.readthedocs.io/en/stable/)
* [https://github.com/dodie/testing-in-bash](https://github.com/dodie/testing-in-bash)
* [https://github.com/bats-core](https://github.com/bats-core)
* [https://github.com/dodie/testing-in-bash](https://github.com/dodie/testing-in-bash)
* [https://marck-oemar.medium.com/unusual-unit-testing-part-1-bash-scripts-with-bats-55ac78e61491](https://marck-oemar.medium.com/unusual-unit-testing-part-1-bash-scripts-with-bats-55ac78e61491) / [https://github.com/marck-oemar/unittesting](https://github.com/marck-oemar/unittesting)

## Debuging

* Make sure to always return a value like `return 0` in each function
* Sometimes we need to use a syntax like below to not run an instruction while the file is being tested by Bats. Another example is adding a `trap`, it seems Bats didn't like that. 

```bash
[[ "$(basename "${0}")" != "bats-exec-test" ]] && concat::__main $*
```

```bash
[[ "$(basename "${0}")" != "bats-exec-test" ]] && trap log::__logDestruct EXIT
```
