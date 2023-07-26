<p align="center"><img src="gophers.png" alt="gophers" style="width: 50%; height: 50%"></p>

# CONFLATE

_Library providing routines to merge and validate JSON, YAML, TOML files and/or structs ([godoc](https://godoc.org/github.com/the4thamigo-uk/conflate))_

_Typical use case: Make your application configuration files **multi-format**, **modular**, **templated**, **sparse**, **location-independent** and **validated**_

[![Build Status](https://secure.travis-ci.org/the4thamigo-uk/conflate.png?branch=master)](https://travis-ci.org/the4thamigo-uk/conflate?branch=master)
[![Coverage Status](https://coveralls.io/repos/the4thamigo-uk/conflate/badge.svg?branch=master&service=github)](https://coveralls.io/github/the4thamigo-uk/conflate?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/the4thamigo-uk/conflate)](https://goreportcard.com/report/github.com/the4thamigo-uk/conflate)

## Description

Conflate is a library and cli-tool, that provides the following features :

* merge data from multiple formats (JSON/YAML/TOML/go structs) and multiple locations (filesystem paths and urls)
* validate the merged data against a JSON schema
* apply any default values defined in a JSON schema to the merged data
* expand environment variables inside the data
* marshal merged data to multiple formats (JSON/YAML/TOML/go structs)

Improvements, ideas and bug fixes are welcomed.

## Background

This project is a hard fork of the original that I developed at https://github.com/miracl/conflate.
However, I will be actively supporting and extending this project independently here.

## Getting started

Run the following command, which will build and install the latest binary in $GOPATH/bin

```
go install  github.com/the4thamigo-uk/conflate/conflate@latest
```
Alternatively, you can install one of the pre-built release binaries from https://github.com/the4thamigo-uk/conflate/releases

## Usage of Library

Please refer to the [godoc](https://godoc.org/github.com/the4thamigo-uk/conflate) and the [example code](./example/main.go)

## Usage of CLI Tool

Help can be obtained in the usual way :

```bash
$conflate --help
Usage of conflate:
  -data value
    	The path/url of JSON/YAML/TOML data, or 'stdin' to read from standard input
  -defaults
    	Apply defaults from schema to data
  -expand
    	Expand environment variables in files
  -format string
    	Output format of the data JSON/YAML/TOML
  -includes string
    	Name of includes array. Blank string suppresses expansion of includes arrays (default "includes")
  -noincludes
    	Switches off conflation of includes. Overrides any --includes setting.
  -schema string
    	The path/url of a JSON v4 schema file
  -validate
    	Validate the data against the schema
  -version
    	Display the version number
```

### Basic Merging of Files by Inclusion

To conflate the following file :

```bash
$cat ./testdata/valid_parent.json
{
  "includes": [
    "valid_child.json",
    "valid_sibling.json"
  ],
  "parent_only" : "parent", 
  "parent_child" : "parent", 
  "parent_sibling" : "parent", 
  "all": "parent"
}
```

...run the following command, which will merge [valid_parent.json](https://raw.githubusercontent.com/the4thamigo-uk/conflate/master/testdata/valid_parent.json), 
[valid_child.json](https://raw.githubusercontent.com/the4thamigo-uk/conflate/master/testdata/valid_child.json), [valid_sibling.json](https://raw.githubusercontent.com/the4thamigo-uk/conflate/master/testdata/valid_sibling.json),
resulting in the file :

```bash
$conflate -data ./testdata/valid_parent.json -format JSON
{
  "all": "parent",
  "child_only": "child",
  "parent_child": "parent",
  "parent_only": "parent",
  "parent_sibling": "parent",
  "sibling_child": "sibling",
  "sibling_only": "sibling"
}
```

Note how the `includes`, are loaded as paths relative to the _including_ file.

Also, note that where the same named value occurs in multiple files, the resulting value follows the following priority rules :
- the values defined in a file, override the values that are imported from the included file(s)
- the values defined in an _included_ file, override the values imported from any included file(s) occurring before it, in the `includes` list

Furthermore, included files can include other files to any depth.

### Choosing an Output Format

To output in a different format use the `-format` option, e.g. TOML :

```bash
$conflate -data ./testdata/valid_parent.json -format TOML
all = "parent"
child_only = "child"
parent_child = "parent"
parent_only = "parent"
parent_sibling = "parent"
sibling_child = "sibling"
sibling_only = "sibling"
```

### Basic Merging of Files Without Inclusion

If you don't want to intrusively embed an `"includes"` array inside your JSON, you can instead provide multiple files on the command line, which are prioritised
from left-to-right, with the right taking highest priority :

```bash
$conflate -data ./testdata/valid_child.json -data ./testdata/valid_sibling.json -format JSON
{
  "all": "sibling",
  "child_only": "child",
  "parent_child": "child",
  "parent_sibling": "sibling",
  "sibling_child": "sibling",
  "sibling_only": "sibling"
}
```

Or alternatively, you can create a top-level file containing only the `includes` array. For fun, lets choose to use YAML for the top-level file, and output TOML :

```bash
$cat toplevel.yaml 
includes:
  - testdata/valid_child.json
  - testdata/valid_sibling.json

$conflate -data toplevel.yaml -format TOML
all = "sibling"
child_only = "child"
parent_child = "child"
parent_sibling = "sibling"
sibling_child = "sibling"
sibling_only = "sibling"
```

### Merging Data from STDIN

If you want to read a file from stdin you use `-data stdin`. For example, here we pipe in some TOML to override a single value in a JSON file :

```bash
$echo 'all="MY OVERRIDDEN VALUE"' |  conflate -data ./testdata/valid_parent.json -data stdin -format JSON
{
  "all": "MY OVERRIDDEN VALUE",
  "child_only": "child",
  "parent_child": "parent",
  "parent_only": "parent",
  "parent_sibling": "parent",
  "sibling_child": "sibling",
  "sibling_only": "sibling"
}
```

Note that in all cases `-data` sources are processed from left-to-right, with values in right files overriding values in left files, so the following doesnt work, since the key `"all"`,
is present in `./testdata/valid_parent.json`:

```bash
$echo 'all="MY OVERRIDDEN VALUE"' |  conflate -data stdin -data ./testdata/valid_parent.json  -format JSON
{
  "all": "parent",
  "child_only": "child",
  "parent_child": "parent",
  "parent_only": "parent",
  "parent_sibling": "parent",
  "sibling_child": "sibling",
  "sibling_only": "sibling"
}
```

### Merging of Remote files by Inclusion

If you instead host a file somewhere else, then just use a URL :

```bash
$conflate -data https://raw.githubusercontent.com/the4thamigo-uk/conflate/master/testdata/valid_parent.json -format JSON
{
  "all": "parent",
  "child_only": "child",
  "parent_child": "parent",
  "parent_only": "parent",
  "parent_sibling": "parent",
  "sibling_child": "sibling",
  "sibling_only": "sibling"
}
```

The `includes` here are also loaded as urls relative the the url of the _including_ file, and follow exactly the same merging rules as for local files.


### Validation Against a JSON Schema

To validate your data against a JSON [schema](https://raw.githubusercontent.com/the4thamigo-uk/conflate/master/testdata/test.schema.json),
use `-schema` in combination with `-validate` :

```bash
$cat ./testdata/blank.yaml

$conflate -data ./testdata/blank.yaml -schema ./testdata/test.schema.json -validate -format YAML
Schema validation failed : The document is not valid against the schema : Invalid type. Expected: object, given: null (#)
```

Note that you can validate any conflated JSON, TOML or JSON files, against the JSON v4 schema, it doesnt have to be pure JSON. 

Also, note that the schema, is itself, validated for correctness against the [JSON v4 meta-schema](https://github.com/the4thamigo-uk/conflate/blob/master/schema.go#L254).

### Use for Simplifying Boiler Plate Validation

One typical use-case for _conflate_, is to use JSON schema validation to avoid having to write lots of boiler plate GO code in order to validate a user's configuration file.
You can perform quite extensive validation using only JSON schemas (even v4 schemas), so this can greatly simplify this kind of work.
One neat approach is to embed your schema into your code as a `const string`, and load it using [NewSchemaData](https://godoc.org/github.com/the4thamigo-uk/conflate#NewSchemaData),
when your app starts up, so you can validate your configuration files against a schema that is directly compiled into your binary.

### Applying Default Values from a JSON Schema

To apply default values defined in a JSON schema use `-schema` in combination with `-defaults`:

```
$conflate -data ./testdata/blank.yaml -schema ./testdata/test.schema.json -defaults -validate -format YAML
all: parent
child_only: child
parent_child: parent
parent_only: parent
parent_sibling: parent
sibling_child: sibling
sibling_only: sibling
```

Note if you also specify `-validate`, the defaults are applied before validation is performed, as you would expect.


### Expansion of Environment Variables

You can optionally expand environment variables in the files like this :

```bash
$export echo MYVALUE="some value"
$export echo MYJSONMAP='{ "item1" : "value1" }'
$echo '{ "my_value": "$MYVALUE", "my_map": $MYJSONMAP }' | conflate -data stdin -expand -format JSON
{
  "my_map": {
    "item1": "value1"
  },
  "my_value": "some value"
}
```

# Acknowledgements

Images derived from originals by Renee French https://golang.org/doc/gopher/
