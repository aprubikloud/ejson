# ejson

`ejson` is a utility for managing a collection of secrets in source control. The
secrets are encrypted using [public
key](http://en.wikipedia.org/wiki/Public-key_cryptography), [elliptic
curve](http://en.wikipedia.org/wiki/Elliptic_curve_cryptography) cryptography
([NaCl](http://nacl.cr.yp.to/) [Box](http://nacl.cr.yp.to/box.html):
[Curve25519](http://en.wikipedia.org/wiki/Curve25519) +
[Salsa20](http://en.wikipedia.org/wiki/Salsa20) +
[Poly1305-AES](http://en.wikipedia.org/wiki/Poly1305-AES)). Secrets are
collected in a JSON file, in which all the string values are encrypted. Public
key(s) are embedded in the file, and the decrypter looks up the corresponding
private key(s) from its local filesystem.

![demo](http://burkelibbey.s3.amazonaws.com/ejson-demo.gif)

The main benefits provided by `ejson` are:

* Secrets can be safely stored in a git repo.
* Changes to secrets are auditable on a line-by-line basis with `git blame`.
* Anyone with git commit access has access to write new secrets.
* Decryption access can easily be locked down to production servers only.
* Secrets change synchronously with application source (as opposed to secrets
  provisioned by Configuration Management).
* Simple, well-tested, easily-auditable source.

NEW:
* A single file can store many versions of the same secrets
* An environment can only decrypt the portion of the secret for which it has the key pair

See [the manpages](https://shopify.github.io/ejson) for more technical documentation.

## Installation

You can download the `.deb` package from [Github Releases](https://github.com/Shopify/ejson/releases).

On development machines (64-bit linux or OS X), the recommended installation
method is via rubygems:

```
gem install ejson
```

## Workflow

### 1: Create the Keydir

By default, EJSON looks for keys in `/opt/ejson/keys`. You can change this by
setting `EJSON_KEYDIR` or passing the `-keydir` option.

```
$ mkdir -p /opt/ejson/keys
```

### 2: Generate a keypair

When called with `-w`, `ejson keygen` will write the keypair into the `keydir`
and print the public key. Without `-w`, it will print both keys to stdout. This
is useful if you have to distribute the key to multiple servers via
configuration management, etc.

```
$ ejson keygen
Public Key:
63ccf05a9492e68e12eeb1c705888aebdcc0080af7e594fc402beb24cce9d14f
Private Key:
75b80b4a693156eb435f4ed2fe397e583f461f09fd99ec2bd1bdef0a56cf6e64
```

```
$ ./ejson keygen -w
53393332c6c7c474af603c078f5696c8fe16677a09a711bba299a6c1c1676a59
$ cat /opt/ejson/keys/5339*
888a4291bef9135729357b8c70e5a62b0bbe104a679d829cdbe56d46a4481aaf
```

### 3: Create an `ejson` file

The format is described in more detail [later on](#format). For now, create a
file that looks something like this. Fill in the `<key>` with whatever you got
back in step 2.

Create this file as `test.ejson`:

```json
{
  "_public_key": "<key>",
  "database_password": "1234password"
}
```

If you wish to use a JSON array, create `test.ejson` like this:

```json
[{
  "_public_key": "<key1>",
  "database_password": "1234password"
}, {
  "_public_key": "<key2>",
  "database_password": "4321password"
}]
```

### 4: Encrypt the file

Running `ejson encrypt test.ejson` will encrypt any new plaintext keys in the
file, and leave any existing encrypted keys untouched:

```json
{
  "_public_key": "63ccf05a9492e68e12eeb1c705888aebdcc0080af7e594fc402beb24cce9d14f",
  "database_password": "EJ[1:WGj2t4znULHT1IRveMEdvvNXqZzNBNMsJ5iZVy6Dvxs=:kA6ekF8ViYR5ZLeSmMXWsdLfWr7wn9qS:fcHQtdt6nqcNOXa97/M278RX6w==]"
}
```

JSON Array:
```json
[{
  "_public_key": "63ccf05a9492e68e12eeb1c705888aebdcc0080af7e594fc402beb24cce9d14f",
  "database_password": "EJ[1:WpvX5/jG+1cU+PjzTDOVlmHkjpFhf+lFo2WWF+RiMGg=:2vIWOra8nFY1ANRdQTz+ZowBmBBV/Vj1:epXERS3pn1VDc4Gi/Zqkh+R2xM1cQFkShLF3Rg==]"
}, {
  "_public_key": "53393332c6c7c474af603c078f5696c8fe16677a09a711bba299a6c1c1676a59",
  "database_password": "EJ[1:irzmtBYYxle/5Z9FpLJHwWcMKl1xqH/UvFrN9nA6KnI=:qrYZxuC7rgpvsoDEP9vxi5TLrb9Iccjy:dRvXLkDNkqjK/02D8m4co6WWpKpC6iejtdLxDA==]"
}]
```

Try adding another plaintext secret to the file and run `ejson encrypt
test.ejson` again. The `database_password` field will not be changed, but the
new secret will be encrypted.

### 5: Decrypt the file

To decrypt the file, you must have a file present in the `keydir` whose name is
the 64-byte hex-encoded public key exactly as embedded in the `ejson` document.
The contents of that file must be the similarly-encoded private key. If you used
`ejson keygen -w`, you've already got this covered.

Unlike `ejson encrypt`, which overwrites the specified files, `ejson decrypt`
only takes one file parameter, and prints the output to `stdout`:

```
$ ejson decrypt foo.ejson
{
  "_public_key": "63ccf05a9492e68e12eeb1c705888aebdcc0080af7e594fc402beb24cce9d14f",
  "database_password": "1234password"
}
```

JSON Array:
```
$ ejson decrypt foo.ejson
[{
  "_public_key": "63ccf05a9492e68e12eeb1c705888aebdcc0080af7e594fc402beb24cce9d14f",
  "database_password": "1234password"
}, {
  "_public_key": "53393332c6c7c474af603c078f5696c8fe16677a09a711bba299a6c1c1676a59",
  "database_password": "4321password"
}]
```

When using JSON arrays, it may be useful to immediately print out the first decrypted json block.
To do this, use the -m flag:
```
$ ejson decrypt -m foo.ejson
{
  "_public_key": "63ccf05a9492e68e12eeb1c705888aebdcc0080af7e594fc402beb24cce9d14f",
  "database_password": "1234password"
}
```

## Format

The `ejson` document format is simple, but there are a few points to be aware
of:

1. It's just JSON.
2. There *must* be a key at the top level of each JSON object named `_public_key`, 
   whose value is a 32-byte hex-encoded (i.e. 64 ASCII byte) public
   key as generated by `ejson keygen`.
3. Any string literal that isn't an object key will be encrypted by default (ie.
   in `{"a": "b"}`, `"b"` will be encrypted, but `"a"` will not.
4. Numbers, booleans, and nulls aren't encrypted.
5. If a key begins with an underscore, its corresponding value will not be
   encrypted. This is used to prevent the `_public_key` field from being
   encrypted, and is useful for implementing metadata schemes.
6. Underscores do not propagate downward. For example, in `{"_a": {"b": "c"}}`,
   `"c"` will be encrypted.

## See also

* If you use Capistrano for deployment you can use [capistrano-ejson](https://github.com/Shopify/capistrano-ejson) to automatically decrypt the secrets on deploy.
