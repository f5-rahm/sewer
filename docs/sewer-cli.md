# Sewer's user command (so many --options!)

Sewer's command line interface, historically named just "sewer" or
"sewer-cli", and implemented in `sewer/cli.py`, is now also available using
the python command line option (eg. `python3 -m sewer`).  In these docs
we'll call it `sewer-cli` in order to avoid already overloaded or generic
names.

The command line tool, however invoked, is still a good vehicle for creating
or renewing a single certificate.  Simple cases may need only a few
--options, but as time goes by the possibilities keep increasing.  The
official doumentation of the options that `sewer-cli` supports remains the
output from running `sewer-cli --help`, but that can be rather terse.  Here
we will discuss what the options are and why they are needed, especially
some recently [or soon-to-be!] added options.

## sewer-cli General Options

`--version`
`--known_providers`
> These are both immediate action options.  They print their information and
exit, ignoring all other arguments (to include argparse errors).

`--log_level`

`--action` "run"|"renew"
> **OBSOLESCENT**.  No longer required in 0.8.3!  Default is "renew".
Has no effect other than changing one word used in one message text.
Whether to create a server key and certificate de novo or reuse the existing
server key (the only thing that CAN be reused) depends only on whether
`--certificate_key` is given.

`--acme_timeout` _seconds_ {7}
> Used to adjust the timeout applied to all requests to the ACME server.
If you need to increase this timeout you'll know it <wink>.
_added by #188 in pre-0.8.3; reworked from #154 from @menduo_

## ACME options

`--endpoint` "production"|"staging"
> Default is "production", viz., issue a legitimate certificate.  Use
"staging" for testing!
_protocol changes enforced since late 2019 for staging are fixed in 0.8.2_

## Account options

To an ACME server, an account is a key pair which has been registered with
that server.  Oh, there's other information that MAY be attached when it is
registered - if you pass you email address to LE you can get timely reminders
about certificates that need to be renewed soon.  But basically, it's that
key pair.

By default, `sewer-cli` will create a new, unique key pair each time it's
run.  And this is okay, because it will also save the key alongside the
certificate and the key that's attested to by the certificate.  But if you
don't want every cert to be issued to a new identity, you'll need to use
`--account_key` to provide the already-registered one.  **NB:** sewer does not
currently provide any way to register, for the first time, a provided key. 
_(this will change in the 0.9 overhaul)_

After the certificate has been created and downloaded, the account key
`sewer-cli` used will be saved alongside the certificate and certificate
key.  After this,the new account key is registered with the ACME endpoint
and may be used for future certificate requests using `--account_key`.

`--account_key` _filepath_
> Filepath to existing, already registered ACME account key.  Default is to
create a new key and register it.

`--email` _email_address_

## Challenge publisher options

`--provider`|`--dns` **name**
> Name of the [DNS] provider to use.
`--dns` is OBSOLESCENT, prefer `--provider` which will be required in 0.9.
As of 0.8.3 this still only supports the legacy DNS providers.

> _ALTERNATE FOR 0.9: make `provider` a required positional parameter,
in accordance with argparse's good advice that
"users expect options to be optional"._

### Driver parameters

During the pre-0.8.3 work, several long options were added to `sewer-cli`
for individual new driver parameters.  These single-use options will be
retired with the release of 0.8.3, as they are all redundant since the
introduction of `--p_opts`.

`--p_opts` name=value ...
>Added late in 0.8.3 development, this will be the only way to pass
parameters into the drivers when 0.8.3 releases.  Like `--alt_domains`,
there can be any number of named parameters following `--p_opts`.

#### Propagation management parameters

There are two sorts of things for which we have to wait: the ACME server
(see `--acme_timeout`) and the service provider (especially DNS propagation
across a global anycast service).  Although these are USED in the core
engine code, they are SET through the driver.  The reasoning is that the
driver is the only part of sewer that might sensibly "know" what sensible
defaults might be for its service provider, and what features are available. 
So eg., although `--prop_timeout` is available to all drivers, legacy DNS
drivers might want to issue a warning if it is specified since (unless
they're updated) they do not support the `prop_timeout` mechanism.

`--p_opts prop_delay=<seconds>`
> Adds a fixed delay after the challenge response have all been setup to
allow the challenge to propagate before any other processing; the default is
no delay.  This is the simplest of the propagation waiting methods, and the
only one available to ~~unmodified~~ minimally modified legacy DNS drivers
such as those in 0.8.3.
_This was `--prop_delay <seconds>` during 0.8.3. development_

`--p_opts prop_timeout=<seconds>`
> Activates the active propagation checks and sets the timeout for that
process; default is to not do these checks.  This requires an implementation
of the driver `unpropagated` method to be useful.  Legacy DNS drivers
inherit a null implementation which always reports _all are ready_ which
short-circuits this process.  _to be added when there's driver support_

`--p_opts prop_sleep_times=<seconds,seconds,...>`
> Comma-separated list of integer number of seconds to sleep after the
first, second, ...  _not all ready_ response from the driver's
`unpropagated` method.  The last value is re-used after the list has been
used up.  Default is "1,2,4,8".  _to be added when there's driver support_

#### DNS driver parameters

`--p_opts alias_domain=<alias_domain_name>`
> Configure an alternate DNS domain in which the challenge responses will be
placed.  See [docs/Aliasing.md](Aliasing) for details.  **Legacy DNS
providers accept this, but require further modification to actually apply
the aliasing that's supported by their parent classes.**
_This was `--alias_domain <name>` during 0.8.3 development.`

## Certificate info

`--domain` **CN-name**
> The primary identity for the certificate.  REQUIRED, no default.  CN-name
is also used to form the default names for a number of files.

`--alt_domains` _SAN-name ..._
> List of alternate identities to be included in the certificate.  Not quite
what pedants would call "SAN", since this should NOT include the CN-name. 
Multiple identities may be given, and sewer-cli will take all parameters
(aka words) as SAN-names until it encounters another option (double-dash). 
Default is an empty list.

`--out_dir` _dirpath_
> Set directory where the certificate and key files will be stored.  Default
is to use the current working directory where sewer was run.

`--certificate_key` _filepath_
> Full path to your [existing] certificate key.  As with the account_key, if
this is not specified, a new key will be created and used (in the
certificate).  Has a similar effect to certbot's `--reuse-key` (sp?) if it
points to the key file from the previous run.

--bundle_name _basename_
> Base name to use for output file, eg., out_dir/basename.{account.key,crt,key}
Default is to use the CN-name
