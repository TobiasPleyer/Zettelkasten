---
date:  2023-10-20
tags:
  - cli
  - howto
  - unix
---

# Pass - The standard UNIX password manager

Beforehand a [[202310202121-gpg-key]] must have been created.

## Usage

```bash
$ pass insert email/work
Enter password for email/work:
Retype password for email/work:
```

Same as above, but let pass generate a random password for us:

```bash
$ pass generate test/example
mkdir: created directory '/home/tobias/.password-store/test'
The generated password for test/example is:
Ktydy3xn,Vv9@O2!ENsg.8Wte
```

Search for passwords via name

```bash
$ pass find email
Search Terms: email
|-- me
|   `-- email
`-- work
    `-- email
```

Store additional metadata in the password:

```bash
$ pass edit work/server/database
```

Search the metadata for text:

```bash
$ pass grep -i couch 2> /dev/null
work/server/database:
CouchDB password
```

```bash
# Load the password into the clipboard
$ pass show -c my/password/path
```

For convenient usage in the shell export environment variables:

```bash
$ export GITHUB_TOKEN=$(pass show -c github/api/token)
```

Or define aliases for this:

```bash
$ alias aws="AWS_ACCESS_KEY_ID=$(pass show aws/access_id) AWS_SECRET_ACCESS_KEY=$(pass show aws/access_token) aws"
```

## Distributed use

Initialize the store as a git repository

```bash
$ pass git init
```

Upload the repo to e.g. Github

```bash
$ pass git push -u origin HEAD
```

The repository can subsequently be cloned to any machine by cloning the git
repository. But in order to use the passwords the keypair needs to be shared.

```bash
$ mkdir ~/exported-keys
$ cd ~/exported-keys
$ gpg --output public.pgp --armor --export tobi.pleyer@gmail.com
$ gpg --output private.pgp --armor --export-secret-key tobi.pleyer@gmail.com
$ scp -r host:exported-keys .
$ gpg --import private.pgp
$ gpg --import public.pgp
```

In order to be able to store new passwords on the second machine the key needs
to be trusted. This can be done like so:

```bash
$ gpg --edit-key tobi.pleyer@gmail.com
gpg> trust
Your decision? 5
```

## Introducing pass

This is taken [from the website | https://www.passwordstore.org/]:

Password management should be simple and follow Unix philosophy. With pass,
each password lives inside of a gpg encrypted file whose filename is the title
of the website or resource that requires the password. These encrypted files
may be organized into meaningful folder hierarchies, copied from computer to
computer, and, in general, manipulated using standard command line file
management utilities.

pass makes managing these individual password files extremely easy. All
passwords live in ~/.password-store, and pass provides some nice commands for
adding, editing, generating, and retrieving passwords. It is a very short and
simple shell script. It's capable of temporarily putting passwords on your
clipboard and tracking password changes using git.

You can edit the password store using ordinary unix shell commands alongside
the pass command. There are no funky file formats or new paradigms to learn.
There is bash completion so that you can simply hit tab to fill in names and
commands, as well as completion for zsh and fish available in the completion
folder. The very active community has produced many impressive clients and GUIs
for other platforms as well as extensions for pass itself.

The pass command is extensively documented in its man page.

## Using the password store

We can list all the existing passwords in the store:

```
zx2c4@laptop ~ $ pass
Password Store
├── Business
│   ├── some-silly-business-site.com
│   └── another-business-site.net
├── Email
│   ├── donenfeld.com
│   └── zx2c4.com
└── France
    ├── bank
    ├── freebox
    └── mobilephone
```

And we can show passwords too:

```
zx2c4@laptop ~ $ pass Email/zx2c4.com
sup3rh4x3rizmynam3
```

Or copy them to the clipboard:

```
zx2c4@laptop ~ $ pass -c Email/zx2c4.com
Copied Email/jason@zx2c4.com to clipboard. Will clear in 45 seconds.
```

There will be a nice password input dialog using the standard gpg-agent (which
can be configured to stay authenticated for several minutes), since all
passwords are encrypted.

We can add existing passwords to the store with insert:

```
zx2c4@laptop ~ $ pass insert Business/cheese-whiz-factory
Enter password for Business/cheese-whiz-factory: omg so much cheese what am i gonna do
```

This also handles multiline passwords or other data with --multiline or -m, and
passwords can be edited in your default text editor using pass edit pass-name.

The utility can generate new passwords using /dev/urandom internally:

```
zx2c4@laptop ~ $ pass generate Email/jasondonenfeld.com 15
The generated password to Email/jasondonenfeld.com is:
$(-QF&Q=IN2nFBx
```

It's possible to generate passwords with no symbols using --no-symbols or -n,
and we can copy it to the clipboard instead of displaying it at the console
using --clip or -c.

And of course, passwords can be removed:

```
zx2c4@laptop ~ $ pass rm Business/cheese-whiz-factory
rm: remove regular file ‘/home/zx2c4/.password-store/Business/cheese-whiz-factory.gpg’? y
removed ‘/home/zx2c4/.password-store/Business/cheese-whiz-factory.gpg’
```

If the password store is a git repository, since each manipulation creates a
git commit, you can synchronize the password store using pass git push and pass
git pull, which call git-push or git-pull on the store.

You can read more examples and more features in the man page.

## Setting it up

To begin, there is a single command to initialize the password store:

```
zx2c4@laptop ~ $ pass init "ZX2C4 Password Storage Key"
mkdir: created directory ‘/home/zx2c4/.password-store’
Password store initialized for ZX2C4 Password Storage Key.
```

Here, ZX2C4 Password Storage Key is the ID of my GPG key. You can use your
standard GPG key or use an alternative one especially for the password store as
shown above. Multiple GPG keys can be specified, for using pass in a team
setting, and different folders can have different GPG keys, by using -p.

We can additionally initialize the password store as a git repository:

```
zx2c4@laptop ~ $ pass git init
Initialized empty Git repository in /home/zx2c4/.password-store/.git/
zx2c4@laptop ~ $ pass git remote add origin kexec.com:pass-store
```

If a git repository is initialized, pass creates a git commit each time the
password store is manipulated.

There is a more detailed initialization example in the man page.
