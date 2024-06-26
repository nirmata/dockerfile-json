# dockerfile-json

Prints Dockerfiles as JSON to stdout, optionally evaluates build args. Uses the [official Dockerfile parser](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/) from buildkit. Plays well with `jq`.

## Contents

- [Contents](#contents)
- [Get it](#get-it)
- [Usage](#usage)
- [Examples](#examples)
  - [JSON output](#json-output)
  - [Extract build stage names](#extract-build-stage-names)
  - [Extract base images](#extract-base-images)
    - [Expand build args, omit stage aliases and `scratch`](#expand-build-args-omit-stage-aliases-and-scratch)
    - [Set build args, omit stage aliases and `scratch`](#set-build-args-omit-stage-aliases-and-scratch)
    - [Expand build args, include all base names](#expand-build-args-include-all-base-names)
    - [Ignore build args, include all base names](#ignore-build-args-include-all-base-names)

## Get it

Using go get:

```bash
go get -u github.com/nirmata/dockerfile-json
```

Or [download the binary for your platform](https://github.com/nirmata/dockerfile-json/releases/latest) from the releases page.

## Usage

### CLI

```text
dockerfile-json [PATHS...]

Usage of dockerfile-json:
  -build-arg value
    	a key/value pair KEY[=VALUE]
  -expand-build-args
    	expand build args (default true)
  -jsonpath string
    	select parts of the output using JSONPath (https://goessner.net/articles/JsonPath)
  -jsonpath-raw
    	when using JSONPath, output raw strings, not JSON values
  -quiet
    	suppress log output (stderr)
```

## Examples

### JSON output

`Dockerfile`
```Dockerfile
ARG ALPINE_TAG=3.10

FROM alpine:$ALPINE_TAG AS build
RUN echo "Hello world" > abc

FROM build AS test
RUN echo "foo" > bar

FROM scratch
COPY --from=build --chown=nobody:nobody abc .
CMD ["echo"]
```

```sh
$ dockerfile-json Dockerfile | jq .
```
```json
{
  "MetaArgs": [
    {
      "Key": "ALPINE_TAG",
      "DefaultValue": "3.10",
      "ProvidedValue": null,
      "Value": "3.10"
    }
  ],
  "Stages": [
    {
      "Name": "build",
      "BaseName": "alpine:3.10",
      "SourceCode": "FROM alpine:${ALPINE_TAG} AS build",
      "Platform": "",
      "Location": [
        {
          "Start": {
            "Line": 3,
            "Character": 0
          },
          "End": {
            "Line": 3,
            "Character": 0
          }
        }
      ],
      "Comment": "",
      "As": "build",
      "From": {
        "Image": "alpine:3.10"
      },
      "Commands": [
        {
          "CmdLine": [
            "echo \"Hello world\" > abc"
          ],
          "Files": null,
          "FlagsUsed": [],
          "Name": "RUN",
          "PrependShell": true
        }
      ]
    },
    {
      "Name": "test",
      "BaseName": "build",
      "SourceCode": "FROM build AS test",
      "Platform": "",
      "Location": [
        {
          "Start": {
            "Line": 6,
            "Character": 0
          },
          "End": {
            "Line": 6,
            "Character": 0
          }
        }
      ],
      "Comment": "",
      "As": "test",
      "From": {
        "Stage": {
          "Named": "build",
          "Index": 0
        }
      },
      "Commands": [
        {
          "CmdLine": [
            "echo \"foo\" > bar"
          ],
          "Files": null,
          "FlagsUsed": [],
          "Name": "RUN",
          "PrependShell": true
        }
      ]
    },
    {
      "Name": "",
      "BaseName": "scratch",
      "SourceCode": "FROM scratch",
      "Platform": "",
      "Location": [
        {
          "Start": {
            "Line": 9,
            "Character": 0
          },
          "End": {
            "Line": 9,
            "Character": 0
          }
        }
      ],
      "Comment": "",
      "From": {
        "Scratch": true
      },
      "Commands": [
        {
          "Chmod": "",
          "Chown": "nobody:nobody",
          "DestPath": ".",
          "From": "build",
          "Name": "COPY",
          "SourceContents": null,
          "SourcePaths": [
            "abc"
          ]
        },
        {
          "CmdLine": [
            "echo"
          ],
          "Files": null,
          "Name": "CMD",
          "PrependShell": false
        }
      ]
    }
  ]
}
```

### Extract build stage names

`Dockerfile`
```Dockerfile
FROM maven:alpine AS build
# ...

FROM build AS test
# ...

FROM openjdk:jre-alpine
# ...
```

```sh
$ dockerfile-json --jsonpath=..As Dockerfile
```
```json
"build"
"test"
```

### Extract base images

`Dockerfile`
```Dockerfile
ARG ALPINE_TAG=3.10
ARG APP_BASE=scratch

FROM alpine:$ALPINE_TAG AS build
# ...

FROM build
# ...

FROM $APP_BASE
# ...
```

#### Expand build args, omit stage aliases and `scratch`

Using `jq`:
```sh
$ dockerfile-json Dockerfile |
    jq '.Stages[] | select(.From | .Stage or .Scratch | not) | .BaseName'
```
```json
"alpine:3.10"
```

Using `--jsonpath`:
```sh

$ dockerfile-json --jsonpath=..Image Dockerfile
```
```json
"alpine:3.10"
```

Using `--jsonpath`, `--jsonpath-raw` output:
```sh
$ dockerfile-json --jsonpath=..Image --jsonpath-raw Dockerfile
```
```json
alpine:3.10
```

#### Set build args, omit stage aliases and `scratch`

```sh
$ dockerfile-json --build-arg ALPINE_TAG=hello-world --jsonpath=..Image Dockerfile
```
```json
"alpine:hello-world"
```

#### Expand build args, include all base names

```sh
$  dockerfile-json --jsonpath=..BaseName Dockerfile
```
```json
"alpine:3.10"
"build"
"scratch"
```

#### Ignore build args, include all base names

```sh
$ dockerfile-json --expand-build-args=false --jsonpath=..BaseName Dockerfile
```
```json
"alpine:${ALPINE_TAG}"
"build"
"${APP_BASE}"
```
