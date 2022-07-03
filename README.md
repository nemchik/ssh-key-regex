# ssh-key-regex

## Purpose

This document is intended to provide regular expressions that can be used to confirm if an SSH public key starts with a valid expected value. To help contribute to this information, please open a pull request.

## Source Information

Most of the information collected and presented here was originally discussed in the comments of a [gist](https://gist.github.com/paranoiq/1932126) posted by [@paranoiq](https://gist.github.com/paranoiq). Specifically, information provided by [@MaPePeR](https://gist.github.com/MaPePeR) was critical in composing the information here.

The [OpenSSH Manual Pages](https://www.openssh.com/manual.html) link to [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen) which lists the valid options for the [-t](https://man.openbsd.org/ssh-keygen#t) flag as:
> dsa | [ecdsa](https://man.openbsd.org/ssh-keygen#ecdsa) | [ecdsa-sk](https://man.openbsd.org/ssh-keygen#ecdsa-sk) | [ed25519](https://man.openbsd.org/ssh-keygen#ed25519) | [ed25519-sk](https://man.openbsd.org/ssh-keygen#ed25519-sk) | [rsa](https://man.openbsd.org/ssh-keygen#rsa)

## Key Generation

Using `ssh-keygen -t ...` for each of the above types as follows:

```shell
ssh-keygen -t dsa
ssh-keygen -t ecdsa
ssh-keygen -t ecdsa-sk
ssh-keygen -t ed25519
ssh-keygen -t ed25519-sk
ssh-keygen -t rsa
```

Results in generation of `*.pub` files for each key type.

## Expected Values

### Description

Within each `*.pub` file, the string of characters at the beginning is expected to be (in order)

1. The key type (this does not always match the value given to the `-t` option when running `ssh-keygen`)
1. A space
1. The base64 value of:
   1. `\0\0\0` (hex characters)
   1. The hex representation of the length of the key type
   1. The key type
1. The rest of the key
1. (optional) A space
1. (optional) A comment describing the key

### Examples

```text
\0\0\0\x0bssh-ed25519
^----------------- \0\0\0 hex characters, always expected
      ^----------- \x0b the hex representation of the length of ssh-ed25519 (11)
          ^------- ssh-ed25519 the key type
```

```text
\0\0\0\x07ssh-rsa
^----------------- \0\0\0 hex characters, always expected
      ^----------- \x07 the hex representation of the length of ssh-rsa (7)
          ^------- ssh-rsa the key type
```

### Shell Output

```shell
$ echo -ne "\0\0\0\x07ssh-dss" | base64
AAAAB3NzaC1kc3M=

$ echo -ne "\0\0\0\x13ecdsa-sha2-nistp256" | base64
AAAAE2VjZHNhLXNoYTItbmlzdHAyNTY=

$ echo -ne "\0\0\0\x22sk-ecdsa-sha2-nistp256@openssh.com" | base64
AAAAInNrLWVjZHNhLXNoYTItbmlzdHAyNTZAb3BlbnNzaC5jb20=

$ echo -ne "\0\0\0\x0bssh-ed25519" | base64
AAAAC3NzaC1lZDI1NTE5

$ echo -ne "\0\0\0\x1ask-ssh-ed25519@openssh.com" | base64
AAAAGnNrLXNzaC1lZDI1NTE5QG9wZW5zc2guY29t

$ echo -ne "\0\0\0\x07ssh-rsa" | base64
AAAAB3NzaC1yc2E=
```

In some of the shell output the final character is `=`, which base64 uses for padding. When the rest of the key is present after the key type in the encoded string it can change the preceding character output in the encoded string.

Using `ssh-rsa` for example:

```shell
$ echo -ne "\0\0\0\x07ssh-rsa" | base64
AAAAB3NzaC1yc2E=

$ echo -ne "\0\0\0\x07ssh-rsa\x00" | base64
AAAAB3NzaC1kc3MA

$ echo -ne "\0\0\0\x07ssh-rsa\xff" | base64
AAAAB3NzaC1kc3P/
```

Because of this, if the base64 output final character is `=` the last 2 characters cannot be reliably used in regular expression matching.

## Regex Strings

```regex
^ssh-dss AAAAB3NzaC1kc3[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
^ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNT[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
^sk-ecdsa-sha2-nistp256@openssh.com AAAAInNrLWVjZHNhLXNoYTItbmlzdHAyNTZAb3BlbnNzaC5jb2[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
^ssh-ed25519 AAAAC3NzaC1lZDI1NTE5[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
^sk-ssh-ed25519@openssh.com AAAAGnNrLXNzaC1lZDI1NTE5QG9wZW5zc2guY29t[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
^ssh-rsa AAAAB3NzaC1yc2[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
```

## Combined Regex

### Support all known key types (less secure)

```regex
^(ssh-dss AAAAB3NzaC1kc3|ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNT|sk-ecdsa-sha2-nistp256@openssh.com AAAAInNrLWVjZHNhLXNoYTItbmlzdHAyNTZAb3BlbnNzaC5jb2|ssh-ed25519 AAAAC3NzaC1lZDI1NTE5|sk-ssh-ed25519@openssh.com AAAAGnNrLXNzaC1lZDI1NTE5QG9wZW5zc2guY29t|ssh-rsa AAAAB3NzaC1yc2)[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
```

`dsa` (`dss`) and `ecdsa`/`sk-ecdsa` may not be considered [secure](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Security).

### Only allow rsa and ed25519/sk-ed25519 (more secure)

```regex
^(ssh-ed25519 AAAAC3NzaC1lZDI1NTE5|sk-ssh-ed25519@openssh.com AAAAGnNrLXNzaC1lZDI1NTE5QG9wZW5zc2guY29t|ssh-rsa AAAAB3NzaC1yc2)[0-9A-Za-z+/]+[=]{0,3}(\s.*)?$
```
