# keepalived

[![License](https://img.shields.io/github/license/voxpupuli/puppet-keepalived.svg)](https://github.com/voxpupuli/puppet-keepalived/blob/master/LICENSE)
[![Build Status](https://travis-ci.org/voxpupuli/puppet-keepalived.png?branch=master)](https://travis-ci.org/voxpupuli/puppet-keepalived)
[![Puppet Forge](https://img.shields.io/puppetforge/v/puppet/keepalived.svg)](https://forge.puppetlabs.com/puppet/keepalived)
[![Puppet Forge - downloads](https://img.shields.io/puppetforge/dt/puppet/keepalived.svg)](https://forge.puppetlabs.com/puppet/keepalived)
[![Puppet Forge - endorsement](https://img.shields.io/puppetforge/e/puppet/keepalived.svg)](https://forge.puppetlabs.com/puppet/keepalived)
[![Puppet Forge - scores](https://img.shields.io/puppetforge/f/puppet/keepalived.svg)](https://forge.puppetlabs.com/puppet/keepalived)

#### Table of Contents

1. [Description](#description)
3. [Usage - Configuration options and additional functionality](#usage)
4. [Limitations - OS compatibility, etc.](#limitations)
5. [Development - Guide for contributing to the module](#development)

## Description

This puppet module manages [keepalived](http://www.keepalived.org/).
The main goal of keepalived is to provide simple and robust facilities for loadbalancing and high-availability to Linux system and Linux based infrastructures.

## Usage

### Basic IP-based VRRP failover

This configuration will fail-over when:

a. Master node is unavailable

```puppet
node /node01/ {
  include keepalived

  keepalived::vrrp::instance { 'VI_50':
    interface         => 'eth1',
    state             => 'MASTER',
    virtual_router_id => '50',
    priority          => '101',
    auth_type         => 'PASS',
    auth_pass         => 'secret',
    virtual_ipaddress => [ '10.0.0.1/29' ],
    track_interface   => ['eth1','tun0'], # optional, monitor these interfaces.
  }
}

node /node02/ {
  include keepalived

  keepalived::vrrp::instance { 'VI_50':
    interface         => 'eth1',
    state             => 'BACKUP',
    virtual_router_id => '50',
    priority          => '100',
    auth_type         => 'PASS',
    auth_pass         => 'secret',
    virtual_ipaddress => [ '10.0.0.1/29' ],
    track_interface   => ['eth1','tun0'], # optional, monitor these interfaces.
  }
}
```

or hiera:

```yaml
---
keepalived::vrrp_instance:
  VI_50:
    interface: 'eth1'
    state: 'MASTER'
    virtual_router_id: '50'
    priority: '101'
    auth_type: 'PASS'
    auth_pass: 'secret'
    virtual_ipaddress: '10.0.0.1/29'
    track_interface:
      - 'eth1'
      - 'tun0'
```

### Add floating routes

```puppet
node /node01/ {
  include keepalived

  keepalived::vrrp::instance { 'VI_50':
    interface         => 'eth1',
    state             => 'MASTER',
    virtual_router_id => '50',
    priority          => '101',
    auth_type         => 'PASS',
    auth_pass         => 'secret',
    virtual_ipaddress => [ '10.0.0.1/29' ],
    virtual_routes    => [ { to  => '168.168.2.0/24', via => '10.0.0.2' },
                           { to  => '168.168.3.0/24', via => '10.0.0.3' } ]
  }
}
```

hiera:

```yaml
---
keepalived::vrrp_instance:
  VI_50:
    interface: 'eth1'
    state: 'MASTER'
    virtual_router_id: '50'
    priority: '101'
    auth_type: 'PASS'
    auth_pass: 'secret'
    virtual_ipaddress: '10.0.0.1/29'
    virtual_routes:
      - to: '168.168.2.0/24'
        via: '10.0.0.2'
      - to: 168.168.3.0/24'
        via: '10.0.0.3'
```

### Detect application level failure

This configuration will fail-over when:

a. NGinX daemon is not running<br>
b. Master node is unavailable

```puppet
node /node01/ {
  include ::keepalived

  keepalived::vrrp::script { 'check_nginx':
    script => '/usr/bin/killall -0 nginx',
  }

  keepalived::vrrp::instance { 'VI_50':
    interface         => 'eth1',
    state             => 'MASTER',
    virtual_router_id => '50',
    priority          => '101',
    auth_type         => 'PASS',
    auth_pass         => 'secret',
    virtual_ipaddress => '10.0.0.1/29',
    track_script      => 'check_nginx',
  }
}

node /node02/ {
  include ::keepalived

  keepalived::vrrp::script { 'check_nginx':
    script => '/usr/bin/killall -0 nginx',
  }

  keepalived::vrrp::instance { 'VI_50':
    interface         => 'eth1',
    state             => 'BACKUP',
    virtual_router_id => '50',
    priority          => '100',
    auth_type         => 'PASS',
    auth_pass         => 'secret',
    virtual_ipaddress => '10.0.0.1/29',
    track_script      => 'check_nginx',
  }
}
```

or hiera:

```yaml
---
keepalived::vrrp_script:
  check_nginx:
    script: '/usr/bin/killall -0 nginx'

keepalived::vrrp_instance:
  VI_50:
    interface: 'eth1'
    state: 'MASTER'
    virtual_router_id: '50'
    priority: '101'
    auth_type: 'PASS'
    auth_pass: 'secret'
    virtual_ipaddress: '10.0.0.1/29'
    track_script: check_nginx
```

### Global definitions

```puppet
class { 'keepalived::global_defs':
  notification_email      => 'no@spam.tld',
  notification_email_from => 'no@spam.tld',
  smtp_server             => 'localhost',
  smtp_connect_timeout    => '60',
  router_id               => 'your_router_instance_id',
}
```

### Soft-restart the Keepalived daemon

```puppet
class { '::keepalived':
  service_restart => 'service keepalived reload',     # When using SysV Init
  # service_restart => 'systemctl reload keepalived', # When using SystemD
}
```

### Opt out of having the service managed by the module

```puppet
class { '::keepalived':
  service_manage => false,
}
```

### Unicast instead of Multicast

**caution: unicast support has only been added to Keepalived since version 1.2.8**

By default Keepalived will use multicast packets to determine failover conditions.
However, in many cloud environments it is not possible to use multicast because of
network restrictions. Keepalived can be configured to use unicast in such environments:

```puppet
  keepalived::vrrp::instance { 'VI_50':
    interface         => 'eth1',
    state             => 'BACKUP',
    virtual_router_id => '50',
    priority          => '100',
    auth_type         => 'PASS',
    auth_pass         => 'secret',
    virtual_ipaddress => '10.0.0.1/29',
    track_script      => 'check_nginx',
    unicast_source_ip => $::ipaddress_eth1,
    unicast_peers     => ['10.0.0.1', '10.0.0.2']
  }
```
The 'unicast\_source\_ip' parameter is optional as Keepalived will bind to the specified interface by default.
The 'unicast\_peers' parameter contains an array of ip addresses that correspond to the failover nodes.


### Creating ip-based virtual server instances with two real servers

This sets up a virtual server www.example.com that directs traffic to
example1.example.com and example2.example.com by matching on an IP address
and port.

```puppet
keepalived::lvs::virtual_server { 'www.example.com':
  ip_address          => '1.2.3.4',
  port                => '80',
  delay_loop          => '7',
  lb_algo             => 'wlc',
  lb_kind             => 'DR',
  persistence_timeout => 86400,
  virtualhost         => 'www.example.com',
  protocol            => 'TCP'
}

keepalived::lvs::real_server { 'example1.example.com':
  virtual_server => 'www.example.com',
  ip_address     => '1.2.3.8',
  port           => '80',
  options        => {
    weight      => '1000',
    'TCP_CHECK' => {
       connection_timeout => '3',
    }
  }
}

keepalived::lvs::real_server { 'example2.example.com':
  virtual_server => 'www.example.com',
  ip_address     => '1.2.3.9',
  port           => '80',
  options        => {
    weight      => '1000',
    'TCP_CHECK' => {
       connection_timeout => '3',
    }
  }
}
```


### Creating firewall mark based virtual server instances with two real servers

This sets up a virtual server www.example.com that directs traffic to
example1.example.com and example2.example.com by matching on a firewall mark
set in iptables or something similar.

```puppet
keepalived::lvs::virtual_server { 'www.example.com':
  fwmark              => '123',
  delay_loop          => '7',
  lb_algo             => 'wlc',
  lb_kind             => 'DR',
  persistence_timeout => 86400,
  virtualhost         => 'www.example.com',
  protocol            => 'TCP'
}

keepalived::lvs::real_server { 'example1.example.com':
  virtual_server => 'www.example.com',
  ip_address     => '1.2.3.8',
  port           => '80',
  options        => {
    weight      => '1000',
    'TCP_CHECK' => {
       connection_timeout => '3',
    }
  }
}

keepalived::lvs::real_server { 'example2.example.com':
  virtual_server => 'www.example.com',
  ip_address     => '1.2.3.9',
  port           => '80',
  options        => {
    weight      => '1000',
    'TCP_CHECK' => {
       connection_timeout => '3',
    }
  }
}
```

## Reference

Reference documentation [coming soon](https://github.com/voxpupuli/puppet-keepalived/issues/158).

## Limitations

Details in `metadata.json`.

## Development

The contributing guide is in [CONTRIBUTING.md](https://github.com/voxpupuli/puppet-keepalived/blob/master/.github/CONTRIBUTING.md).

## Release Notes/Contributors/Etc.

Details in `CHANGELOG.md`.

Migrated from https://github.com/arioch/puppet-keepalived to Vox Pupuli.

