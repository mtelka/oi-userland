# Support for Perl modules

As of August 2022 OpenIndiana contains ca 220 third party Perl modules and the
number is expected to grow in future.  That's nice but it could cause a
maintenance nightmare considering that:

1. every module needs to be rebuild when we add support for new major version of Perl, while upstream releases new major Perl version usually once a year,
2. every module needs to be rebuild when we remove support for obsoleted major version of Perl, while upstream obsoletes major Perl version usually once a year,
3. a new module version needs to be re-integrated when we upgrade to newer module's upstream version,
4. when support for new module is being added, many repeating tasks needs to be done and this is error prone.

To avoid the nightmare we developed set of tools that allows to do much of the
work automatically or semi-automatically.  The set contains following tools
found in the `tools` directory:

* `perl-integrate-module`
* `perl-meta-deps`
* `perl-version-convert`

The `perl-integrate-module` script is main tool used for all tasks related to
Perl module integration.  It is used for initial module integration, upgrade of
a module to newer version, rebuild of a module to either support new Perl
version, or to obsolete support for old Perl version, or to rebuild it for any
other reason.

The `perl-meta-deps` tool is used internally by oi-userland build framework to
gather Perl module dependencies and it is not expected to be used directly by
package maintainers.

The `perl-version-convert` tool is used for conversion of Perl module versions
to pkg(7)-compatible versions.

## Perl modules basics

Perl modules are named using a word-like sequences separated by double-colon
`::`, for example
[Mojolicious::Command](https://metacpan.org/pod/Mojolicious::Command),
[Mojo](https://metacpan.org/pod/Mojo), or
[File::chdir](https://metacpan.org/pod/File::chdir) are all Perl modules.  One
or more Perl modules are bundled together to form **distribution**.
Distribution names are word-like sequences separated by hyphen `-`, for example
[Mojolicious](https://metacpan.org/dist/Mojolicious) or
[File-chdir](https://metacpan.org/dist/File-chdir) are distributions.  The
[Mojolicious](https://metacpan.org/dist/Mojolicious) distribution contains
[Mojolicious::Command](https://metacpan.org/pod/Mojolicious::Command),
[Mojo](https://metacpan.org/pod/Mojo) and many other Perl modules, while the
[File-chdir](https://metacpan.org/dist/File-chdir) distribution contains single
[File::chdir](https://metacpan.org/pod/File::chdir) Perl module.

Distributions are versioned.  For example, `9.26` is version of
[Mojolicious](https://metacpan.org/dist/Mojolicious) released on
[2022-05-23](https://metacpan.org/release/SRI/Mojolicious-9.26), and `0.05` is
version of [Cwd-Guard](https://metacpan.org/dist/Cwd-Guard) released on
[2016-04-14](https://metacpan.org/release/KAZEBURO/Cwd-Guard-0.05).

The distinction between `module` and `distribution` is sometimes blurry and it
is often found that word `module` is used instead of proper `distribution`.
This could be also case in this document.  We warned you :-).

Our tools expects that Perl modules are distributed via
[CPAN](https://www.cpan.org/)/[MetaCPAN](https://metacpan.org/).

All Perl modules integrated into oi-userland lives in the
`components/perl/DISTRIBUTION` directory, where `DISTRIBUTION` is the
distribution name.  All files in the `components/perl/DISTRIBUTION` directory
are created automatically by the `perl-integrate-module` script with few
exceptions:

* optional `perl-integrate-module.conf` file which is a configuration file for the `perl-integrate-module` script,
* any patches living in the `patches` subdirectory.

Please note that most modules integrate properly even without any patches
and/or `perl-integrate-module.conf` file.

Once published, every integrated module produces a package with
`library/perl-5/NAME` fmri, where `NAME` is the distribution name converted to
lower case.  This package is just a meta-package used to bring-in real packages
with support for particular Perl versions.  Such packages are named
`library/perl-5/NAME-PERLVER`, where `PERLVER` is major Perl version with
stripped dot, for example `536` for Perl 5.36, and there are usually two such
packages, because we usually support two major Perl versions.

	$ pkg list -a 'library/perl-5/mojolicious*' | grep -v 'o$'
	NAME (PUBLISHER)                                  VERSION                    IFO
	library/perl-5/mojolicious                        9.26-2022.0.0.1            i--
	library/perl-5/mojolicious-534                    9.26-2022.0.0.1            i--
	library/perl-5/mojolicious-536                    9.26-2022.0.0.1            i--
	$

## Perl module versions

Perl modules and Perl itself uses [unusual versioning
scheme](https://dev.to/grinnz/a-guide-to-versions-in-perl-3ng1) where there are
common leading zeros or other funny things.  Version ordering and/or comparison
is thus non-trivial.

This is incomaptible with strict pkg(7) versioning scheme so we created
`perl-version-convert` tool to convert Perl module versions to
pkg(7)-compatible versions.  The tool usually simply converts some known Perl
version patterns, but it also contains some exceptions or special handling for
some modules.  The special handling is needed either for historical reasons, or
because the module uses so unusual versioning that we had to do something with
it.

Some examples:

	$ tools/perl-version-convert Mojolicious 9.26
	9.26
	$ tools/perl-version-convert Cwd-Guard 0.05
	0.5
	$ tools/perl-version-convert Authen-PAM 0.16
	16
	$ tools/perl-version-convert Mozilla-CA 20211001
	2021.10.1
	$ tools/perl-version-convert File-chdir 0.1011
	0.10.11
	$ tools/perl-version-convert Module-Build 0.4231
	0.4231
	$

## How to integrate new Perl module

To integrate new Perl module you should follow these steps:

1. Make sure your system is reasonably up-to-date.

	If you are unsure, or do not know, just run `pkg update`, then `reboot`.  After that the `pkg list -u` output should be empty.

2. Install all supported Perl versions: 

		pkg install -v 'runtime/perl*'

3. Install all Perl modules:

		pkg list -H 'runtime/perl-*' \
			| sed -e 's|^runtime/perl-\([0-9]*\).*$|\1|g' \
			| sort -u \
			| while read PERLVER ; do
				pkg install -v 'library/perl-5/*-'$PERLVER
			done

4. Make sure the module you aim to integrate is _not_ in the repo yet.

	First, set `MODULE` environment variable to contain the module name you
	want to integrate.  For example, if you want to integrate module `Foo::Bar` do
	this:

		MODULE=Foo::Bar

	Then run the following command.  Basically, its output should be empty:

		pkg search "*/${MODULE//::/\/}.pm"

	Example for modules `DateTime::TimeZone`, `FindBin`, and
	`DateTime::TimeZone::LMT`:

		$ MODULE=DateTime::TimeZone ; pkg search "*/${MODULE//::/\/}.pm"
		INDEX      ACTION VALUE                                           PACKAGE
		path       file   usr/perl5/vendor_perl/5.34/DateTime/TimeZone.pm pkg:/library/perl-5/datetime-timezone-534@2.52-2022.0.0.0
		path       file   usr/perl5/vendor_perl/5.36/DateTime/TimeZone.pm pkg:/library/perl-5/datetime-timezone-536@2.52-2022.0.0.0
		$ MODULE=FindBin ; pkg search "*/${MODULE//::/\/}.pm"
		INDEX      ACTION VALUE                         PACKAGE
		path       file   usr/perl5/5.22/lib/FindBin.pm pkg:/runtime/perl-522@5.22.4-2022.0.0.3
		path       file   usr/perl5/5.24/lib/FindBin.pm pkg:/runtime/perl-524@5.24.3-2020.0.1.3
		path       file   usr/perl5/5.34/lib/FindBin.pm pkg:/runtime/perl-534@5.34.1-2022.0.0.1
		path       file   usr/perl5/5.36/lib/FindBin.pm pkg:/runtime/perl-536@5.36.0-2022.0.0.0
		$ MODULE=DateTime::TimeZone::LMT ; pkg search "*/${MODULE//::/\/}.pm"
		$

	This means the only module that is not provided by any package yet is
	`DateTime::TimeZone::LMT`.  Both `DateTime::TimeZone` and `FindBin` Perl
	modules are already provided by some packages and you do not need to integrate
	them again.  If you would try to continue with `DateTime::TimeZone` you will
	just rebuild the existing `DateTime::TimeZone` module.  An attempt to integrate
	`FindBin` will likely lead to non-installable packages.

5. Determine distribution for the module.

	You can determine the distribution manually by looking at
	[CPAN](https://www.cpan.org/)/[MetaCPAN](https://metacpan.org/), but it is far
	simpler to do it using the following command:

		DIST=$(curl -s https://fastapi.metacpan.org/v1/module/$MODULE | jq -r '.distribution')

	Example:

		$ MODULE=Mojo ; DIST=$(curl -s https://fastapi.metacpan.org/v1/module/$MODULE | jq -r '.distribution') ; echo $DIST
		Mojolicious
		$

6. Find master module for distribution.

	Many distributions contains single module only, while the module name
	and the distribution name match.  The **master module** is such single module.
	For example [File::chdir](https://metacpan.org/pod/File::chdir) module is
	**master module** in [File-chdir](https://metacpan.org/dist/File-chdir)
	distribution.

	A lot of distributions contains more than one module, while they
	contain a module with name that is matching the distribution name.  Such module
	is **master module**.  For example, [Mojo](https://metacpan.org/pod/Mojo) or
	[Mojolicious::Command](https://metacpan.org/pod/Mojolicious::Command) modules
	are _not_ **master modules** in
	[Mojolicious](https://metacpan.org/dist/Mojolicious) distribution.  The
	**master module** in [Mojolicious](https://metacpan.org/dist/Mojolicious)
	distribution is [Mojolicious](https://metacpan.org/pod/Mojolicious) module.

	To handle both above cases you could simply do this:

		MODULE=${DIST//-/::}

	The last set of distributions and the only distributions where finding
	proper **master module** is really important are distributions _without_ a
	module that matches the distribution name.  For such distributions you need to
	pick one module that represents the whole distribution.  Sometimes it is easy
	and straightforward, for example [LWP](https://metacpan.org/pod/LWP) module is
	obvious **master module** for
	[libwww-perl](https://metacpan.org/dist/libwww-perl) distribution.  But
	sometimes it could be harder, like with
	[HTML-Element-Extended](https://metacpan.org/dist/HTML-Element-Extended)
	distribution where we opted for
	[HTML::ElementTable](https://metacpan.org/pod/HTML::ElementTable) as **master
	module**.  Fortunately, these cases are rare.  Once you found the **master
	module** assign it manually to the `MODULE` environment variable.

	The **master module** is the module you are going to integrate.  In the
	next steps when we refer to **module** we always mean **master module**.

7. Check distribution versions.

	You should make sure our build framework properly understands
	distribution's versioning scheme.  You should do the check using the
	`perl-version-convert` tool:

		tools/perl-version-convert $DIST VERSION

	It is important to make sure all versions during the whole distribution
	history are understood properly because some other modules could require any
	version of your distribution.  For example, if your distribution `Foo` used to
	use versions like `0.20021028` (i.e. containing date) in past and they switched
	to something like `1.09`, `1.10`, etc. ten years ago, you still need to make
	sure the old versioning scheme is understood and converted properly by the
	`perl-version-convert` tool.  Otherwise, in a case there is some distribution
	`Bar` that would require `Foo` and it would be happy with an old version of
	`Foo` then `Bar` could have in its metadata someting like `requires "Foo":
	0.20021028` which will lead either to failure during integration of `Bar`, or
	to wrong dependencies of `Bar` packages making them possibly not installable.

	To do the check you could look at the distribution version history at
	[CPAN](https://www.cpan.org/)/[MetaCPAN](https://metacpan.org/).  You'll
	usually find that the distribution uses single or highly compatible versioning
	scheme during the lifetime of the distribution.  For example
	[Mojolicious](https://metacpan.org/dist/Mojolicious) and
	[File-chdir](https://metacpan.org/dist/File-chdir) distributioins are such
	distributions with nice versioning schemes.

	Example of version check for
	[Mojolicious](https://metacpan.org/dist/Mojolicious) and
	[File-chdir](https://metacpan.org/dist/File-chdir) distributioins:

		$ tools/perl-version-convert Mojolicious 9.26
		9.26
		$ tools/perl-version-convert Mojolicious 9.09
		9.9
		$ tools/perl-version-convert Mojolicious 9.0
		9.0
		$ tools/perl-version-convert File-chdir 0.1011
		0.10.11
		$ tools/perl-version-convert File-chdir 0.10
		0.10
		$ tools/perl-version-convert File-chdir 0.01
		0.1
		$

	In a case the distribution uses not yet supported versioning scheme, or
	its versioning scheme was changed during the distribution lifetime, or it is
	otherwise strange, you might need to update the `perl-version-convert` tool to
	understand the distribution's versioning.

	Here is an example of changing versioning scheme we opted to adapt
	`perl-version-convert` for to get nicer pkg(7) versions:

		$ tools/perl-version-convert Email-Sender 2.500
		2.5
		$ tools/perl-version-convert Email-Sender 1.500
		1.5
		$ tools/perl-version-convert Email-Sender 1.300035
		1.3.35
		$ tools/perl-version-convert Email-Sender 0.120002
		0.120.2
		$ tools/perl-version-convert Email-Sender 0.093380
		0.93.380
		$ tools/perl-version-convert Email-Sender 0.004
		0.4
		$

		$ pkg list -afv library/perl-5/email-sender
		FMRI                                                                         IFO
		pkg://openindiana.org/library/perl-5/email-sender@2.5-2022.0.0.0:20220802T142549Z ---
		pkg://openindiana.org/library/perl-5/email-sender@1.3.21-2022.0.0.0:20220611T211315Z ---
		$

	Without the change in `perl-version-convert` we would get versions like
	`2.500` and `1.300.21` for `library/perl-5/email-sender` package instead.

	An example of unsupported version format:

		$ tools/perl-version-convert Net-DNS-Resolver-Mock 1.20220817
		ERROR: Unsupported version format: 1.20220817
		UNSUPPORTED_VERSION_FORMAT_1.20220817
		$

8. Prepare your local git repo.

	You should make sure you start with up-to-date `oi/hipster` branch when
	creating a branch for your new Perl module integration:

		git checkout oi/hipster
		git pull
		git checkout -b $DIST oi/hipster

	Set `WS_TOOLS`:

		WS_TOOLS=$(git rev-parse --show-toplevel)/tools

9. Run `perl-integrate-module`.

	Simply run `$WS_TOOLS/perl-integrate-module $MODULE`.  The tool will
	try to automatically integrate your module and create git changeset for it.

10. Asses the `perl-integrate-module` results.

	First, you should look at the `perl-integrate-module` output.  If there
	was something wrong or unexpected the `perl-integrate-module` tool printed
	warning(s), error(s), or fatal error on `stderr`.  If there is neither `ERROR`
	nor `FATAL` in the output and all possible `WARNING`s looks acceptable for you
	then you should review the created git changeset using the `git show` command.
	If you are happy with the changeset then the integration was successful and you
	should proceed to [next step](#next-step).

	`FATAL` error prevents successful run of the `perl-integrate-module`
	tool and causes immediate failure of the integration.  You should fix the
	problem before you retry the `perl-integrate-module` run.  Typical `FATAL`
	error is when there are some packages missing and you need to install them.
	When the `perl-integrate-module` tool ends with `FATAL` error it leave
	uncommitted changes hanging in git for analysis and/or review.  The subsequent
	`perl-integrate-module` run will cleanup these remnants.

	When `perl-integrate-module` prints `ERROR` it means the integration
	failed and the resulting changeset will lack something important depending what
	exactly failed.  The `ERROR` itself won't cause integration failure and the
	process will continue.  You need to fix all `ERROR`s otherwise the module
	integration won't be correct.  To fix most `ERROR`s you'll probably need to
	create module specific `perl-integrate-module.conf` configuration file.

	`WARNING`s are something you need to review to make sure the
	integration went correctly and the resulting changeset is really okay.  To
	resolve some `WARNING`s you would need to create module specific
	`perl-integrate-module.conf` configuration file, while other `WARNING`s could
	be safely ignored.

	In any case, you should never manually modify any file created by the
	`perl-integrate-module` tool in the `components/perl/DISTRIBUTION` directory.
	Such changes will be lost.  The only files in the
	`components/perl/DISTRIBUTION` directory that are never modified by the
	`perl-integrate-module` script and you are allowed and sometimes expected to
	manually create and/or modify are:

	* patches in the `patches` subdirectory, and
	* `perl-integrate-module.conf` configuration file.


---

TODO (unsorted notes)

---


3a. run `$WS_TOOLS/perl-integrate-module MODULE` (where `MODULE` is `Clone` in this case)

3b. if the run failed (`ERROR` or `FATAL` reported by the `perl-integrate-module` script, or you are not satisfied with the result), then you need custom `perl-integrate-module.conf`; if the integration is okay (you are satisfied with the `git show` output), go to 3h.

3c. make sure there is nothing staged for git (use `git status` to confirm that).  In a case the `perl-integrate-module` script failed with `FATAL` you'll probably find that there is something staged, so you need to cleanup using `git reset --hard`.

3d. create (or modify, if this is your 2nd or 3rd or ... iteration) `perl-integrate-module.conf` in the newly created directory for the perl module (in this case you need to create/modify `components/perl/Clone/perl-integrate-module.conf`).

3e. add `perl-integrate-module.conf` to git (`git add components/perl/Clone/perl-integrate-module.conf` then `git commit -m "perl-integrate-module.conf for Clone"`) - make sure the changeset contains `components/perl/Clone/perl-integrate-module.conf` only and nothing else

3f. remove the changeset with `Add Clone perl module` message from your git history (`git rebase -i HEAD~2` should usually do the right thing); if this is your 2nd or 3rd or ... iteration you'll probably need `git rebase -i HEAD~3` - in this case I suggest to squash all `perl-integrate-module.conf` commits to one.

3g. go to 3a

3h. squash your `perl-integrate-module.conf` commit(s) to the `Add Clone perl module` commit: `git rebase -i HEAD~2` then move the `pick ... perl-integrate-module.conf for Clone` line below the `pick ... Add Clone perl module` line and change `pick ... perl-integrate-module.conf for Clone` to `f ... perl-integrate-module.conf for Clone`.

3i. continue to 4 (see #9062)

4. review what was automatically created using `git show` - it should be okay (I checked it), you do not need to manually change anything.
5. push your new Razor2::Client::Agent integration to github: `git push -v ...`
6. create new PR.

---

There are three supported `include` tags: `%include-1%` (this will go right before `shared-macros.mk` include in the final `Makefile`), `%include-2%` (right before `common.mk` include in the final `Makefile`), and `%include-3%` (right after `common.mk` include in the final `Makefile`).
