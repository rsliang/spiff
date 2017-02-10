```
                                        ___ _ __ (_)/ _|/ _|
                                       / __| '_ \| | |_| |_
                                       \__ \ |_) | |  _|  _|
                                       |___/ .__/|_|_| |_|
                                           |_|

```

---

**NOTE**: *Active development on spiff is currently paused, including Pull Requests.  Very severe issues will be addressed, and we will still be actively responding to requests for help via Issues.*

---

spiff is a command line tool and declarative YAML templating system, specially designed for generating BOSH deployment manifests.

Contents:
- [Installation](#installation)
- [Usage](#usage)
- [dynaml Templating Tanguage](#dynaml-templating-language)
	- [(( foo ))](#-foo-)
	- [(( foo.bar.[1].baz ))](#-foobar1baz-)
	- [(( "foo" ))](#-foo--1)
	- [(( "foo" bar ))](#-foo-bar-)
	- [(( auto ))](#-auto-)
	- [(( merge ))](#-merge-)
		- [<<: (( merge ))](#--merge-)
			- [merging maps](#merging-maps)
			- [merging lists](#merging-lists)
	- [(( a || b ))](#-a--b-)
	- [(( !foo ))](#-foo--2)
	- [(( static_ips(0, 1, 3) ))](#-static_ips0-1-3-)


# Installation

Official release executable binaries can be downloaded via [Github releases](https://github.com/cloudfoundry-incubator/spiff/releases) for Darwin and Linux machines (and virtual machines).

Some of spiff's dependencies have changed since the last official release, and spiff will not be updated to keep up with these dependencies.  Working dependencies are vendored in the `Godeps` directory (more information on the `godep` tool is available [here](https://github.com/tools/godep)).  As such, trying to `go get` spiff will likely fail; the only supported way to use spiff is to use an official binary release.

# Usage

### `spiff merge template.yml [template2.ymll ...]`

Merge a bunch of template files into one manifest, printing it out.

See 'dynaml templating language' for details of the template file, or examples/ subdir for more complicated examples.

Example:

```
spiff merge cf-release/templates/cf-deployment.yml my-cloud-stub.yml
```

### `spiff diff manifest.yml other-manifest.yml`

Show structural differences between two deployment manifests.

Unlike basic diffing tools and even `bosh diff`, this command has semantic 
knowledge of a deployment manifest, and is not just text-based. For example,
if two manifests are the same except they have some jobs listed in different
orders, `spiff diff` will detect this, since job order matters in a manifest.
On the other hand, if two manifests differ only in the order of their
resource pools, for instance, then it will yield and empty diff since 
resource pool order doesn't actually matter for a deployment.

Also unlike `bosh diff`, this command doesn't modify either file.

It's tailored for checking differences between one deployment and the next.

Typical flow:

```sh
$ spiff merge template.yml [templates...] > upgrade.yml
$ bosh download manifest [deployment] current.yml
$ spiff diff upgrade.yml current.yml
$ bosh deployment upgrade.yml
$ bosh deploy
```


# dynaml Templating Language

Spiff uses a declarative, logic-free templating language called 'dynaml'
(dynamic yaml).

Every dynaml node is guaranteed to resolve to a YAML node. It is *not*
string interpolation. This keeps developers from having to think about how
a value will render in the resulting template.

A dynaml node appears in the .yml file as an expression surrounded by two
parentheses. They can be used as the value of a map or an entry in a list.

The following is a complete list of dynaml expressions:


## `(( foo ))`

Look for the nearest 'foo' key (i.e. lexical scoping) in the current
template and bring it in.

e.g.:

```yaml
fizz:
  buzz:
    foo: 1
    bar: (( foo ))
  bar: (( foo ))
foo: 3
bar: (( foo ))
```

This example will resolve to:

```yaml
fizz:
  buzz:
    foo: 1
    bar: 1
  bar: 3
foo: 3
bar: 3
```

The following will not resolve because the key name is the same as the value to be merged in:
```yaml
foo: 1

hi:
  foo: (( foo ))
```

## `(( foo.bar.[1].baz ))`

Look for the nearest 'foo' key, and from there follow through to .bar.baz.

A path is a sequence of steps separated by dots. A step is either a word for
maps, or digits surrounded by brackets for list indexing.

If the path cannot be resolved, this evaluates to nil. A reference node at the
top level cannot evaluate to nil; the template will be considered not fully
resolved. If a reference is expected to sometimes not be provided, it should be
used in combination with '||' (see below) to guarantee resolution.

Note that references are always within the template, and order does not matter.
You can refer to another dynamic node and presume it's resolved, and the
reference node will just eventually resolve once the dependent node resolves.

e.g.:

```yaml
properties:
  foo: (( something.from.the.stub ))
  something: (( merge ))
```

This will resolve as long as 'something' is resolveable, and as long as it
brings in something like this:

```yaml
from:
  the:
    stub: foo
```

## `(( "foo" ))`

String literal. The only escape character handled currently is '"'.

## `(( "foo" bar ))`

Concatenation (where bar is another dynaml expr).

e.g.

```yaml
domain: example.com
uri: (( "https://" domain ))
```

In this example `uri` will resolve to the value `"https://example.com"`.

## `(( auto ))`

Context-sensitive automatic value calculation.

In a resource pool's 'size' attribute, this means calculate based on the total
instances of all jobs that declare themselves to be in the current resource
pool.

e.g.:

```yaml
resource_pools:
  - name: mypool
    size: (( auto ))

jobs:
  - name: myjob
    resource_pool: mypool
    instances: 2
  - name: myotherjob
    resource_pool: mypool
    instances: 3
  - name: yetanotherjob
    resource_pool: otherpool
    instances: 3
```

In this case the resource pool size will resolve to '5'.

## `(( merge ))`

Bring the current path in from the stub files that are being merged in.

e.g.:

```yaml
foo:
  bar:
    baz: (( merge ))
```

Will try to bring in `foo.bar.baz` from the first stub, or the second, etc.,
returning the value from the first stub that provides it.

If the corresponding value is not defined, it will return nil. This then has the
same semantics as reference expressions; a nil merge is an unresolved template.
See `||`.

### `<<: (( merge ))`

#### Merging maps

**values.yml**
```yaml
foo:
  a: 1
  b: 2
```

**template.yml**
```yaml
foo:
  <<: (( merge ))
  b: 3
  c: 4
```

`spiff merge template.yml values.yml` yields:

```yaml
foo:
  a: 1
  b: 2
  c: 4
```

#### Merging lists

**values.yml**
```yaml
foo:
  - 1
  - 2
```

**template.yml**
```yaml
foo:
  - 3
  - <<: (( merge ))
  - 4
```

`spiff merge template.yml values.yml` yields:

```yaml
foo:
  - 3
  - 1
  - 2
  - 4
```

## `(( a || b ))`

Uses a, or b if a cannot be resolved.

e.g.:

```yaml
foo:
  bar:
    - name: some
    - name: complicated
    - name: structure

mything:
  complicated_structure: (( merge || foo.bar ))
```

This will try to merge in `mything.complicated_structure`, or, if it cannot be
merged in, use the default specified in `foo.bar`.

## `(( !foo ))`

Leaves the string `(( !foo ))` in the resulting merge.

e.g.: calling `spiff merge` on a file with the contents below:

```yaml
bar: (( !foo ))
foo: 33
```

will simply result in:

```yaml
bar: (( !foo ))
foo: 33
```

## `(( static_ips(0, 1, 3) ))`

Generate a list of static IPs for a job.

e.g.:

```yaml
jobs:
  - name: myjob
    instances: 2
    networks:
    - name: mynetwork
      static_ips: (( static_ips(0, 3, 4) ))
```

This will create 3 IPs from `mynetwork`s subnet, and return two entries, as
there are only two instances. The two entries will be the 0th and 3rd offsets
from the static IP ranges defined by the network.

For example, given the file bye.yml:

```yaml
networks: (( merge ))

jobs:
  - name: myjob
    instances: 3
    networks:
    - name: cf1
      static_ips: (( static_ips(0,3,60) ))
```

and file hi.yml:

```yaml
networks:
- name: cf1
  subnets:
  - cloud_properties:
      security_groups:
      - cf-0-vpc-c461c7a1
      subnet: subnet-e845bab1
    dns:
    - 10.60.3.2
    gateway: 10.60.3.1
    name: default_unused
    range: 10.60.3.0/24
    reserved:
    - 10.60.3.2 - 10.60.3.9
    static:
    - 10.60.3.10 - 10.60.3.70
  type: manual
```

```
spiff merge bye.yml hi.yml
```

returns


```yaml
jobs:
- instances: 3
  name: myjob
  networks:
  - name: cf1
    static_ips:
    - 10.60.3.10
    - 10.60.3.13
    - 10.60.3.70
networks:
- name: cf1
  subnets:
  - cloud_properties:
      security_groups:
      - cf-0-vpc-c461c7a1
      subnet: subnet-e845bab1
    dns:
    - 10.60.3.2
    gateway: 10.60.3.1
    name: default_unused
    range: 10.60.3.0/24
    reserved:
    - 10.60.3.2 - 10.60.3.9
    static:
    - 10.60.3.10 - 10.60.3.70
  type: manual
```
.

If bye.yml was instead

```yaml
networks: (( merge ))

jobs:
  - name: myjob
    instances: 2
    networks:
    - name: cf1
      static_ips: (( static_ips(0,3,60) ))
```

```
spiff merge bye.yml hi.yml
```

instead returns

```yaml
jobs:
- instances: 2
  name: myjob
  networks:
  - name: cf1
    static_ips:
    - 10.60.3.10
    - 10.60.3.13
networks:
- name: cf1
  subnets:
  - cloud_properties:
      security_groups:
      - cf-0-vpc-c461c7a1
      subnet: subnet-e845bab1
    dns:
    - 10.60.3.2
    gateway: 10.60.3.1
    name: default_unused
    range: 10.60.3.0/24
    reserved:
    - 10.60.3.2 - 10.60.3.9
    static:
    - 10.60.3.10 - 10.60.3.70
  type: manual
```
