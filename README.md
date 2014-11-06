puppet
======
[![Build Status](https://travis-ci.org/ocf/puppet.svg)](https://travis-ci.org/ocf/puppet)

This repository contains the [Puppet][puppet] modules used to maintain and
configure the servers and desktops used by the [Open Computing Facility][ocf]
at UC Berkeley.

These modules are generally intended to be used on the latest [Debian][debian]
stable release, though probably also work on Debian-derived distros (such as
[Ubuntu][ubuntu]).

This README outlines development practices for OCF volunteer staff members. If
you're a member of the UC Berkeley community and interested in getting
involved, [check us out][hello]!

## Making and testing changes
### Your puppet environment

Puppet environments are stored on the puppetmaster in `/opt/puppet/env/`. Each
user should have their own environment with the same name as their user name:

    ckuehl@lightning:~$ ls -l /opt/puppet/env
    drwxr-xr-x 6 ckuehl  ocf  4.0K Nov  5 11:04 ckuehl
    drwxr-xr-x 5 daradib ocf  4.0K Aug 11 22:53 daradib
    drwxr-xr-x 5 tzhu    ocf  4.0K Sep  2 17:46 tzhu
    drwxr-xr-x 6 willh   ocf  4.0K Oct  9 13:56 willh

The puppetmaster has service CNAME `puppet`, so you can connect to it via `ssh
puppet`.

Each environment is just a copy of this git repository owned by the respective
user. You should make your changes here and push them to GitHub to be deployed
into production.

If creating a new environment, you should either copy an existing environment,
or clone the repo and remember to run `git submodule update --init`.

### Testing using your puppet environment

Before pushing, you should test your changes by switching at least one of the
affected servers to your puppet environment and triggering a run. We store node
definitions in LDAP, so changing a server's environment requires a `/admin`
Kerberos principal (and corresponding LDAP privileges).

Start by getting a ticket for your `/admin` principal and launching ldapvi on
the server's LDAP entry:

    ckuehl@supernova:~$ kinit ckuehl/admin ldapvi cn=raptors

A text representation of the LDAP entry will open in your editor. For example:

    0 cn=raptors,ou=Hosts,dc=OCF,dc=Berkeley,dc=EDU
    cn: raptors
    type: server
    ipHostNumber: 169.229.10.200
    objectClass: device
    objectClass: ocfDevice
    environment: production

Change the environment from `production` to your own, then save the file and
quit your editor. `ldapvi` will ask you to type `y` to confirm the changes.

After switching the environment, you can connect to the server, trigger a
puppet run, and watch the logs:

    ckuehl@supernova:~$ ssh raptors
    ckuehl@raptors:~$ sudo puppet-trigger
    ckuehl@raptors:~$ sudo less +F /var/log/syslog

Make sure to switch the environment back to production after pushing your
changes.

### Linting and validating the puppet config

Our puppet config is pretty big; we have a handy script which:

* Parses puppet manifests for syntax errors
* Validates Ruby `erb` templates for syntax errors
* Lints puppet manifests to ensure a consistent style

To run it, just execute `scripts/validate.py` (located in the repo). For best
results, run it on the puppetmaster.

### Deploying changes to production

GitHub is the authoritative source for this repository; at all times, the
`production` environment on the puppetmaster will be a clone of the `master`
branch on GitHub (we use [a webhook][webhook] to keep it up-to-date).

Pushing to GitHub will immediately update the production environment, but your
changes will not take effect until the puppet agent runs on each server (every
30 minutes, at an arbitrary offset). You can use the `puppet-trigger` script if
you want it to happen faster.

## Conventions and styling
### Naming conventions

All OCF modules that are primarily intended for OCF use (currently, all of
them) should be prefixed with `ocf_`.

For modules that apply only to a specific service (such as the MySQL server),
use the service CNAME (such as `mysql`) for the module name. Otherwise, use
common sense to come up with a reasonable name (e.g. `ocf_desktop` or
`ocf_ssl`).

For manifests that should apply to all (or most) OCF servers, such as one that
sets up LDAP/Kerberos authentication, consider just creating a new class under
the `common` module.

Try not to refer to servers by hostname (such as `lightning`). Instead, use the
service CNAME (such as `puppet`) or the top-level variables `$::hostname` and
`$::fqdn`.

### Including third-party modules

Third-party modules can be helpful. Try to only use ones that are actively
maintained.

We use [git submodules][sobmodules] to include third-party modules in our
config. This has benefits over storing them in a global directory on the
puppetmaster (e.g. with the `puppet module` tool):

* This puppet config repository is self-contained
* Adding and updating modules can be tested in an environment before being
  inflicted on every server
* Staff members can test third-party modules without needing root on the
  puppetmaster

### Styling

In lieu of an actual style guide, please try to make your code consistent with
the existing code (or help write a style guide?), and ensure that it passes
validation (including linting).

### Minimal config file management

Try to change as few things as possible; this makes upgrading to newer versions
of packages and operating systems easier, as well as making it more obvious to
future staffers what options you actually changed.

Instead of overwriting an entire config file just to change one value, try to
[use augeas][augeas] ([example][augeas-example]) or [sed][sed]
([example][sed-example]) to change just the necessary values.

## Future improvements

* Finish renaming and cleanup of OCF's modules
* Consider using [librarian-puppet][lib-puppet] instead of git submodules
* Trigger puppet runs automatically after production is updated
* Better monitoring of puppet runs (e.g. to see when a server has not updated
  recently, which is a common problem on desktops)

[puppet]: https://en.wikipedia.org/wiki/Puppet_(software)
[ocf]: https://www.ocf.berkeley.edu/
[debian]: https://www.debian.org/
[ubuntu]: http://www.ubuntu.com/
[hello]: https://hello.ocf.berkeley.edu/
[webhook]: https://github.com/ocf/puppet/blob/master/modules/ocf_puppet/files/webhook/github.cgi
[sobmodules]: http://git-scm.com/book/en/v2/Git-Tools-Submodules
[lib-puppet]: http://librarian-puppet.com/
[augeas]: http://projects.puppetlabs.com/projects/1/wiki/puppet_augeas
[augeas-example]: https://github.com/ocf/puppet/blob/master/modules/common/manifests/auth.pp#L75
[sed]: http://projects.puppetlabs.com/projects/puppet/wiki/Simple_Text_Patterns/5
[sed-example]: https://github.com/ocf/puppet/blob/jessie-desktops/modules/desktop/manifests/grub.pp#L13
