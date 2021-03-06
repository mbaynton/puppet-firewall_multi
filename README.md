# firewall_multi

[![Build Status](https://img.shields.io/travis/alexharv074/puppet-firewall_multi.svg)](https://travis-ci.org/alexharv074/puppet-firewall_multi)

## Overview

The `firewall_multi` module provides a defined type wrapper for spawning [puppetlabs/firewall](https://github.com/puppetlabs/puppetlabs-firewall) resources for arrays of certain inputs.

At present the following inputs can be arrays:
* source
* destination
* proto
* icmp
* provider

## Version compatibility

Each release of the firewall_multi module is compatible with a specific release of puppetlabs-firewall, starting at firewall v1.8.0. Earlier versions of the firewall module are not supported.

firewall_multi|firewall
--------------|--------
earlier|1.8.0
1.7.0|1.8.0
1.7.0|1.8.1
1.8.0|1.8.2
1.9.0|1.9.0
1.10.0|1.10.0
1.10.0|1.11.0

## Usage

It is expected that a standard set up for the firewall module is followed, in particular with respect to the purging of firewall resources.  If a user of this module, for instance, removes addresses from an array of sources, the corresponding firewall resources will only be removed if purging is enabled.  This might be surprising to the user in a way that impacts security.

Otherwise, usage of the firewall_multi defined type is the same as with the firewall custom type, the only exceptions being that some parameters optionally accept arrays.

## Parameters

* `source`: the source IP address or network or an array of sources.  Use of this parameter causes a firewall resource to be spawned for each address in the array of sources, and a string like 'from *x.x.x.x/x*' to be appened to each spawned resource's title to guarantee uniqueness in the catalog.  If not specified, a default of undef is used and the resultant firewall resource provider will not be passed a source.

* `destination`: the destination IP address or network or an array of destinations.  Use of this parameter causes a firewall resource to be spawned for each address in the array of destinations, and a string like 'to *y.y.y.y/y*' to be appended to each spawned resource's title to guarantee uniqueness in the catalog.  If not specified, a default of undef is used and the resultant firewall resource provider will not be passed a destination.

* `proto`: the protocol or an array of protocols.  Use of this parameter causes a firewall resource to be spawned for each protocol in the array of protocols, and a string like 'protocol *aa*' to be appended to each spawned resource's title to guarantee uniqueness in the catalog.  If not specified, a default of undef is used and the resultant firewall resource provider will not be passed a protocol.

* `icmp`: the ICMP type or an array of ICMP types specified as an array of integers or strings.  Use of this parameter causes a firewall resource to be spawned for each type in the array of ICMP types, and a string like 'icmp type *nn*' to be appended to each spawned resource's title to guarantee uniqueness in the catalog.  If not specified, a default of undef is used and the resultant firewall resource provider will not be passed an ICMP type.

* `provider`: the provider to use, either `iptables` or `ip6tables`.

* Any other parameter accepted by firewall is also accepted and set for each firewall resource created without error-checking.

## Examples

### Array of sources

```puppet
firewall_multi { '100 allow http and https access':
  source => [
    '10.0.10.0/24',
    '10.0.12.0/24',
    '10.1.1.128',
  ],
  dport  => [80, 443],
  proto  => tcp,
  action => accept,
}
```

This will cause three resources to be created:

* Firewall['100 allow http and https access from 10.0.10.0/24']
* Firewall['100 allow http and https access from 10.0.12.0/24']
* Firewall['100 allow http and https access from 10.1.1.128']

### Arrays of sources and destinations

```puppet
firewall_multi { '100 allow http and https access':
  source => [
    '10.0.10.0/24',
    '10.0.12.0/24',
  ],
  destination => [
    '10.2.0.0/24',
    '10.3.0.0/24',
  ],
  dport  => [80, 443],
  proto  => tcp,
  action => accept,
}
```

This will cause four resources to be created:

* Firewall['100 allow http and https access from 10.0.10.0/24 to 10.2.0.0/24']
* Firewall['100 allow http and https access from 10.0.10.0/24 to 10.3.0.0/24']
* Firewall['100 allow http and https access from 10.0.12.0/24 to 10.2.0.0/24']
* Firewall['100 allow http and https access from 10.0.12.0/24 to 10.3.0.0/24']

### Array of protocols

```puppet
firewall_multi { '100 allow DNS lookups':
  dport  => 53,
  proto  => ['tcp', 'udp'],
  action => 'accept',
}
```

This will cause two resources to be created:

* Firewall['100 allow DNS lookups protocol tcp']
* Firewall['100 allow DNS lookups protocol udp']

### Array of ICMP types

```puppet
firewall_multi { '100 accept icmp output':
  chain  => 'OUTPUT',
  proto  => 'icmp',
  action => 'accept',
  icmp   => [0, 8],
}
```

This will cause two resources to be created:

* Firewall['100 accept icmp output icmp type 0']
* Firewall['100 accept icmp output icmp type 8']

### Array of providers

Open a firewall for IPv4 and IPv6 on a web server:

```puppet
firewall_multi { '100 allow http and https access':
  dport    => [80, 443],
  proto    => 'tcp',
  action   => 'accept',
  provider => ['ip6tables', 'iptables'],
}
```

### Used in place of a single firewall resource

If none of firewall_multi's array functionality is used, then the firewall_multi and firewall resources can be used interchangeably.

### Use with Hiera and create_resources

Some users may prefer to externalise the firewall resources in Hiera and use the `create_resources` function:

```yaml
---
myclass::firewall_multis:
  '00099 accept tcp port 22 for ssh':
    dport: '22'
    action: 'accept'
    proto: 'tcp'
    source:
      - 10.0.0.3/32
      - 10.10.0.0/26
```

Meanwhile we would have manifest code that looks something like this:

Puppet 3.x:

```puppet
class myclass (
  $firewall_multis,
) {
  validate_hash($firewall_multis)
  create_resources(firewall_multi, $firewall_multis)
  ...
}
```

Puppet 4.x:

```puppet
class myclass (
  Hash $firewall_multis,
) {
  create_resources(firewall_multi, $firewall_multis)
  ...
}
```

### The alias lookup

Users who wish to externalise the firewall resources in Hiera should be aware of a feature that was added to Hiera in version 3, namely the [alias lookup function](https://docs.puppet.com/hiera/3.0/variables.html#the-alias-lookup-function), which makes it possible to define networks as arrays in Hiera and then look these up from within the `firewall_multi` definitions.

The following examples show how to do that:

```yaml
---
mylocaldomains:
  - 10.0.0.3/32
  - 10.10.0.0/26
myotherdomains:
  - 172.0.1.0/26

myclass::firewall_multis:
  '00099 accept tcp port 22 for ssh':
    dport: '22'
    action: 'accept'
    proto: 'tcp'
    source: "%{alias('mylocaldomains')}"
  '00200 accept tcp port 80 for http':
    dport: '80'
    action: 'accept'
    proto: 'tcp'
    source: "%{alias('myotherdomains')}"
```

## Known Issues

If you are using Puppet 3.x please understand the implications of [Issue #5](https://github.com/alexharv074/puppet-firewall_multi/issues/5).

This module does not sanity-check the proposed inputs for the resultant firewall resources.  We assume that we can rely on the firewall resource types themselves to detect invalid inputs.

## Development

Please read CONTRIBUTING.md before contributing.

### Testing

Make sure you have:

* rake
* bundler

Install the necessary gems:

    bundle install

To run the tests from the root of the source code:

    bundle exec rake spec

To run the acceptance tests:

    BEAKER_set=centos-72-x64 bundle exec rspec spec/acceptance

### Release

This module uses Puppet Blacksmith to publish to the Puppet Forge.

Ensure you have these lines in `~/.bash_profile`:

    export BLACKSMITH_FORGE_URL=https://forgeapi.puppetlabs.com
    export BLACKSMITH_FORGE_USERNAME=alexharvey
    export BLACKSMITH_FORGE_PASSWORD=xxxxxxxxx

Build the module:

    bundle exec rake build

Push to Forge:

    bundle exec rake module:push

Clean the pkg dir (otherwise Blacksmith will try to push old copies to Forge next time you run it and it will fail):

    bundle exec rake module:clean


