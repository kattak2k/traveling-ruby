# Traveling Ruby: self-contained, portable Ruby binaries

![](https://openclipart.org/image/300px/svg_to_png/181225/Travel_backpacks.png)

Traveling Ruby is a project which supplies self-contained, "portable" Ruby binaries: Ruby binaries that can run on any Linux distribution and any OS X machine. Ruby appp developers can bundle these binaries with their Ruby app so that they can distribute a single package to end users, without needing end users to first install Ruby or gems.

## Motivation

Ruby is one of our favorite programming langugages. Most people use it for web development, but Ruby is so much more. We at Phusion have been using Ruby for years for writing sysadmin automation scripts, developer command line tools and more. [Heroku's Toolbelt](https://toolbelt.heroku.com/) and [Chef](https://www.chef.io/) have also demonstrated that Ruby is an excellent language for these sorts of things.

However, distributing such Ruby apps to inexperienced end users or non-Ruby-programmer end users is problematic. If users have to install Ruby first, or if they have to use RubyGems, [they can easily run into problems](#end_user_problems). Even if they already have Ruby installed, they [can still run into problems](#end_user_problems), e.g. by having the wrong Ruby version installed. The point is, it's a very real problem that [could harm your reputation](#end_user_problems).

One solution is to build OS-specific installation packages, e.g. DEBs, RPMs, .pkgs, etc. However, this has two disadvantages:

 1. It requires a lot of work. You not only have to build separate packages for each OS, but also each OS *version*. An in the context of Linux, you have to treat each distribution as another OS, further increasing the number of combinations. Suppose that you want to support ~2 versions of CentOS/RHEL, ~2 versions of Debian, ~3 versions of Ubuntu, ~2 recent OS X releases, on both x86 and x86_64. You'll have to create `(2 + 2 + 3) * 2 + 2 = 16` packages.
 2. Because you typically cannot build an OS-specific installation package using anything but that OS, you need heavyweight tooling, e.g. a fleet of VMs. For example, you can only build Ubuntu 14.04 DEBs on Ubuntu 14.04; you cannot build them from your OS X developer laptop.

This is exactly the approach that Chef has chosen. They built [Omnibus](https://github.com/opscode/omnibus), an automation system which spawns an army of VMs for building platform-specific packages. It works, but it's heavyweight and a big hassle. You need a big build machine for that if you want to have reasonable build time. And be prepared to make 20 cups of coffee.

But there is another -- much simpler -- solution.

### Way of the Traveling Ruby

The solution that Traveling Ruby advocates, is to distribute your app as a single self-contained tar.gz/zip package that already includes a precompiled Ruby interpreter for a specific platform (that the Traveling Ruby project provides), as well as all gems that your app depends on. This eliminates the need for heavyweight tooling:

 * A tar.gz/zip file can be created on any platform using small and simple tools.
 * You can create packages for any OS, regardless of which OS you are using.

This makes the release process much simpler. Instead of having to create more than 10 packages using a fleet of VMs, you just create 3 packages quickly and easily from your developer laptop. These 3 packages cover all the major platforms that your end users are on:

 * Linux x86.
 * Linux x86_64.
 * OS X.

(Note: Traveling Ruby doesn't bother with supporting Windows yet. Sorry.)

However, distributing a precompiled Ruby interpreter that works for all end users, is more easily said than done. [Read this section](#why_precompiled_binary_difficult) to learn why it's difficult.

Traveling Ruby aims to solve the problem of supplying precompiled Ruby binaries that work for all end users.

## Getting started

Here's a 5 minute tutorial that shows you how it works.

Suppose you have a simple hello world Ruby app:

    $ mkdir hello_app
    $ cd hello_app
    $ echo 'puts "hello world"' > hello.rb
    $ ruby hello.rb
    hello world

You want to package this and distribute it to end users.

### Preparation

Let's start by installing the Traveling Ruby SDK:

    gem install traveling-ruby

The next step is to prepare packages for all the target platforms, by creating a directory each platform, and by copying your app into each directory.

    $ mkdir hello-1.0.0-linux-x86
    $ mkdir hello-1.0.0-linux-x86/app
    $ cp hello.rb hello-1.0.0-linux-x86/app/

    $ mkdir hello-1.0.0-linux-x86_64
    $ mkdir hello-1.0.0-linux-x86_64/app
    $ cp hello.rb hello-1.0.0-linux-x86_64/app/

    $ mkdir hello-1.0.0-osx
    $ mkdir hello-1.0.0-osx/app
    $ cp hello.rb hello-1.0.0-osx/app/

Next, use the `traveling-ruby extract <PLATFORM> <DIRECTORY>` command to download binaries for each platform, and to extract them into each directory.

    $ traveling-ruby extract linux-x86 hello-1.0.0-linux-x86/ruby
    $ traveling-ruby extract linux-x86_64 hello-1.0.0-linux-x86_64/ruby
    $ traveling-ruby extract osx hello-1.0.0-osx/ruby

Now, each package directory will have Ruby binaries included. It looks like this:
Your directory structure will now look like this:

    hello_app/
     |
     +-- hello.rb
     |
     +-- hello-1.0.0-linux-x86/
     |   |
     |   +-- app/
     |   |   |
     |   |   +-- hello.rb
     |   |
     |   +-- ruby/
     |       |
     |       +-- bin/
     |       |   |
     |       |   +-- ruby
     |       |   +-- ...
     |       +-- ...
     |
     +-- hello-1.0.0-linux-x86_64/
     |   |
     |  ...
     |
     +-- hello-1.0.0-osx/
         |
        ...

### Quick sanity testing

Let's do a basic sanity test by running your app with a bundled Ruby interpreter. Suppose that you are developing on OS X. Run this:

    $ cd hello-1.0.0-osx
    $ ./bin/ruby app/hello.rb
    hello world
    $ cd ..

### Creating a wrapper script

Now that you've verified that the bundled Ruby interpreter works, you'll want create a *wrapper script*. After all, you don't want your users to run `/path-to-your-app/bin/ruby /path-to-your-app/app/hello.rb`. You want them to run `/path-to-your-app/hello`.

Here's what a wrapper script could look like:

    #!/bin/bash
    set -e

    # Figure out where this script is located.
    SELFDIR="`dirname \"$0\"`"
    SELFDIR="`cd \"$SELFDIR\" && pwd`"

    # Run the actuall app using the bundled Ruby interpreter.
    exec "$SELFDIR/bin/ruby" "$SELFDIR/app/hello.rb"

Save this file `wrapper.sh` in your project's root directory. Then you can copy it to each of your package directories:

    $ chmod +x wrapper.sh
    $ cp wrapper.sh hello-1.0.0-linux-x86/hello
    $ cp wrapper.sh hello-1.0.0-linux-x86_64/hello
    $ cp wrapper.sh hello-1.0.0-osx/hello

### Finalizing packages

Your package directories are now ready. You can finalize the packages by packaging up all these directories using tar:

    $ tar -czf hello-1.0.0-linux-x86.tar.gz hello-1.0.0-linux-x86
    $ tar -czf hello-1.0.0-linux-x86_64.tar.gz hello-1.0.0-linux-x86_64
    $ tar -czf hello-1.0.0-osx.tar.gz hello-1.0.0-osx
    $ rm -rf hello-1.0.0-linux-x86
    $ rm -rf hello-1.0.0-linux-x86_64
    $ rm -rf hello-1.0.0-osx

Congratulations, you have created packages using Traveling Ruby!

An x86 Linux user could now use your app like this:

 1. The user downloads `hello-1.0.0-linux-x86.tar.gz`.
 2. The user extracts this file.
 3. The user runs your app:

         $ /path-to/hello-1.0.0-linux-x86/hello
         hello world

## Gems and dependencies

Your app can depend on any gem you wish, subject to the [limitations](#limitations). You must include the gems in your packages, and your wrapper script must pass the right parameters to the Ruby interpreter in order for your gems to be found. It is recommended that you manage gems using Bundler.

For example, suppose that you want to use the [https://github.com/davetron5000/methadone](Methadone) gem. Start by creating a Gemfile:

    source 'https://rubygems.org'
    gem 'methadone'

Next, install your gem bundle into some *local directory*, because during the packaging phase we'll want to copy all the files over.

    $ bundle install --path vendor

Then create your package directories and extract Traveling Ruby binaries into them, as was described in "Getting started".

    $ mkdir hello-1.0.0-linux-x86
    ...etc...
    $ mkdir hello-1.0.0-linux-x86_64
    ...etc...
    $ mkdir hello-1.0.0-osx
    ...etc...

Copy over your Bundler gem directory into the packages:

    $ cp -pR vendor hello-1.0.0-linux-x86/
    $ cp -pR vendor hello-1.0.0-linux-x86_64/
    $ cp -pR vendor hello-1.0.0-osx/

Copy over your Gemfile and Gemfile.lock into each gem directory inside the packages:

    $ cp Gemfile Gemfile.lock hello-1.0.0-linux-x86/vendor/
    $ cp Gemfile Gemfile.lock hello-1.0.0-linux-x86_64/vendor/
    $ cp Gemfile Gemfile.lock hello-1.0.0-osx/vendor/

Create a Bundler config file in each of the gem directories inside the packages. This Bundler config file tells Bundler that gems are to be found in the same directory that the Gemfile resides in.

    $ editor bundler-config
    BUNDLE_PATH: .
    BUNDLE_DISABLE_SHARED_GEMS: '1'

    $ mkdir hello-1.0.0-linux-x86/vendor/.bundle
    $ mkdir hello-1.0.0-linux-x86_64/vendor/.bundle
    $ mkdir hello-1.0.0-osx/vendor/.bundle

    $ cp bundler-config hello-1.0.0-linux-x86/vendor/.bundle/config
    $ cp bundler-config hello-1.0.0-linux-x86_64/vendor/.bundle/config
    $ cp bundler-config hello-1.0.0-osx/vendor/.bundle/config

Create a wrapper script `wrapper.sh` that tells Bundler where your Gemfile is (and where the gems are), and executes your app using Bundler:

    #!/bin/bash
    set -e

    # Figure out where this script is located.
    SELFDIR="`dirname \"$0\"`"
    SELFDIR="`cd \"$SELFDIR\" && pwd`"

    # Run the actuall app using the bundled Ruby interpreter.
    export BUNDLE_GEMFILE="$SELFDIR/vendor"
    exec "$SELFDIR/bin/ruby" -rbundler/setup "$SELFDIR/app/hello.rb"

Copy over this new wrapper script to each of your package directories and finalize the packages:

    $ chmod +x wrapper.sh
    $ cp wrapper.sh hello-1.0.0-linux-x86/hello
    $ cp wrapper.sh hello-1.0.0-linux-x86_64/hello
    $ cp wrapper.sh hello-1.0.0-osx/hello
    
    $ tar -czf hello-1.0.0-linux-x86.tar.gz hello-1.0.0-linux-x86
    $ tar -czf hello-1.0.0-linux-x86_64.tar.gz hello-1.0.0-linux-x86_64
    $ tar -czf hello-1.0.0-osx.tar.gz hello-1.0.0-osx
    $ rm -rf hello-1.0.0-linux-x86
    $ rm -rf hello-1.0.0-linux-x86_64
    $ rm -rf hello-1.0.0-osx

<a name="limitations"></a>

## Limitations

To be written.

 * You cannot use gems with native extensions. Well technically you can, but then you have to compile for 3 different platforms. I haven't documented this yet.

## The Traveling Ruby build system

To be written.

## FAQ

<a name="why_precompiled_binary_difficult"></a>

### Why it is difficult to supply a precompiled Ruby interpreter that works for all end users?

Chances are that you think that you can compile a Ruby binary on a certain OS, and that users using that same OS can use your Ruby binary. Not quite. Not even when they run the same OS *version* as you do.

Basically, there are two problems that can prevent a binary from working on another system:

 1. Libraries that your binary depends on, may not be available on the user's OS.
    * When compiling Ruby, you might accidentally introduce a dependency on a non-standard library! As a developer you probably have all sorts non-standard libraries installed on your system. While compiling Ruby, the Ruby build system autodetects certain libraries and links to them.
    * Even different versions of the same OS ship with different libraries! You cannot count on a certain library from an older OS version, to be still available on a newer version of the same OS.
 2. On Linux, there are issues with glibc symbols. This is a little more complicated, so read on.

Assuming that your binary doesn't use *any* libraries besides the C standard library, binaries compiled on a newer Linux system usually do not work on an older Linux system, even if you do not use newer APIs. This is because of glibc symbols. Each function in glibc - or symbol as C/C++ programmers call it - actually has multiple versions. This allows the glibc developers to change the behavior of a function without breaking backwards compatibility with apps that happen to rely on bugs or implementation-specific behavior. During the linking phase, the linker "helpfully" links against the most recent version of the symbol. The thing is, glibc introduces new symbol versions very often, resulting in binaries that will most likely depend on a recent glibc.

There is no way to tell the compiler and linker to use older symbol versions unless you want to manually specify the version for each and every symbol, which is an undoable task.

The only sane way to get around the glibc symbol problem, and to prevent accidental linking to unwanted libraries, is to create a tightly controlled build environment. On Linux, this build environment with come with an old glibc version. This tightly controlled build environment is sometimes called a "holy build box".

The Traveling Ruby project provides such a holy build box.

#### Why not just statically link the Ruby binary?

First of all: easier said than done. The compiler prefers to link to dynamic libraries. You have to hand-edit lots of Makefiles to make everything properly link statically. You can't just add `-static` as compiler flag and expect everything to work.

Second: Ruby is incompatible with static linking. On Linux systems, executables which are statically linked to the C library cannot dynamically load shared libraries. Yet Ruby extensions are shared libraries, and a Ruby interpreter that cannot load Ruby extensions is heavily crippled.

So in Traveling Ruby we've taken a different approach. Our Ruby binaries are dynamically linked against the C library, but only uses old symbols to avoid glibc symbol problems. We also ship carefully-compiled versions of dependent shared libraries, like OpenSSL, ncurses, libedit, etc.

<a name="end_user_problems"></a>

### Why is it problematic for end users if I don't bundle a Ruby interpreter?

First of all, users just want to run your app as quickly as possible. Requiring them to install Ruby first is not only a distraction, but it can also cause problems. Here are a few examples of such problems:

 * There are various ways to install Ruby, e.g. by compiling from source, by using `apt-get` and `yum`, by using RVM/rbenv/chruby, etc. The choices are obvious to us, but users could get confused by the sheer number of choices. Worse: not all choices are good. APT and YUM often provide old versions of Ruby, which may not be the one that you want. Compiling from source and using rbenv/chruby requires the user to have a compiler toolchain and appropriate libraries pre-installed. How should they know what to pre-install before they can install Ruby? The Internet is filled with a ton of old and outdated tutorials, further increasing their confusion.
 * Users could install Ruby incorrectly, e.g. to a location that isn't in PATH. They could then struggle with "command not found" errors. PATH is obvious to us, but there are a lot of users out there can barely use the command line. We shouldn't punish them for lack of knowledge, they are end users after all.

One way to solve this is for you to "hold the user's hand", by going through the trouble of supplying 4 or 5 different platform-specific installation instructions for installing Ruby. These instructions must be continuously kept up-to-date. That's a lot of work and QA on your part, and I'm sure you just want to concentrate on building your app.

And let's for the sake of argument suppose that the user somehow has Ruby correctly installed. They still need to install your app. The most obvious way to do that is through RubyGems. But that will open a whole new can of worms:

 * On some OSes, RubyGems is configured in such a way that the RubyGems-installed commands are not in PATH. For a classic example, try running this on Debian 6:

        $ sudo apt-get install rubygems
        $ sudo gem install rails
        $ rails new foo
        bash: rails: command not found

   Not a good first impression for end users.
 * Depending on how Ruby is installed, you may or may not have to run `gem install` with `sudo`. It depends on whether `GEM_HOME` is writable by the current user or not. You can't tell them "always run with sudo", because if their `GEM_HOME` is in their home directory, running `gem install` with sudo will mess up all sorts of permissions.
 * Did I just mention `sudo`? No, because `sudo` by default resets a lot of environment variables. Environment variables which may be important for Ruby to work.
   - If the user installed Ruby with RVM, then the user has to run `rvmsudo` instead of sudo. RVM is implemented by setting `PATH`, `RUBYLIB`, `GEM_HOME` and other environment variables. rvmsudo is a wrapper around sudo which preserves these environment variables.
   - If the user installed Ruby with rbenv or chruby... pray that they know what they're doing. Rbenv and chruby also require correct `PATH`, `RUBYLIB`, `GEM_HOME` etc to be set to specific values, but they provide no rvmsudo-like tool for preserving them after taking sudo access. So if you want to be user-friendly, you have to write documentation that tells users to sudo to a bash shell first, fix their `PATH`, `RUBYLIB` etc, and *then* run `gem install`.

The point is, there's a lot of opportunity for end users to get stuck, confused and frustrated. You can deal with all these problems by supplying excellent documentation that handles all of these cases (and probably more, because there are infinite ways to break things). That's exactly what we've done for [Phusion Passenger](https://www.phusionpassenger.com). Our [RubyGems installation instructions](https://www.phusionpassenger.com/documentation/Users%20guide%20Nginx.html#rubygems_generic_install) spell out exactly how to install Ruby for each of the major operating systems, how to find out whether they need to run `gem install` with sudo, how to find out whether they need to run rvmsudo instead of sudo. It has been a lot of work, and even then we still haven't covered all the cases. We're still lacking documentation on what rbenv and chruby users should do. Right now, rbenv/chruby users regularly contact our community discussion forum about installation problems related to sudo access and environment variables.

Or you can just use Traveling Ruby and be done with it. We can't do it for Phusion Passenger because by its very nature it has to work with an already-installed Ruby, but maybe you can for writing your next command line tool.

#### The problems sound hypothetical. Is it really that big of a deal for end users?

Yes. These problems can put off your users from installing your app at all and can give you a bad reputation. Especially Chef has suffered a lot from this. A lot of people have had bad experience in the past with installing Chef through RubyGems. Chef has solved this problem for years by supplying platform-specific packages for years (DEBs, RPMs, etc), but the reputation stuck: there are still people out there who shun Chef because they think they have to install Ruby and use RubyGems.

#### I target OS X, which already ships Ruby. Should I still bundle a Ruby interpreter?

Yes. OS X versions up to 10.8 Mountain Lion ship Ruby 1.8. Only starting from 10.9 Mavericks does it ship Ruby 2.0. There are significant compatibility differences between Ruby 1.8 and 2.0. Future OS X versions might ship yet another Ruby version. Only by bundling Ruby can you be sure that OS upgrades won't break your app.
