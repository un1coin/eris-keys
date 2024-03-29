# eris-keys

A simple tool for generating keys, producing and verifying signatures.

Features:
- basic support for ECDSA (secp256k1) and Shnorr (ed25519)
- command-line and http interfaces
- password based encryption (AES-GCM) with time locks
- addressing schemes and key naming

## WARNING: This is semi-audited cryptographic software. It should not yet be presumed safe.  It should probably be replaced by a wrapper around gpg. The HTTP server binds localhost but does not (yet) use CORS.

The code is mostly a fork of go-ethereum/crypto. Major changes include removing support for other ECDSA curves,
adding support for ED25519, and using AES-GCM for encryption. And of course the pretty cli and http interfaces :)

# Install

eris-keys supports the same key/signature implementation used by bitcoin and ethereum, but [it's a C-library](https://github.com/bitcoin/secp256k1) that depends on `gmp` for big number arithmetic.
On Mac you should be able to `brew install gmp`, on ubuntu `sudo apt-get install libgmp3-dev`.

Then

```
go install github.com/eris-ltd/eris-keys
```


# CLI

The cli works over an http server. To start the server run `eris-keys server &`

## Generate a key, sign something, verify the signature

```
> ADDR=`eris-keys gen --no-pass`
> PUB=`eris-keys pub --addr $ADDR`
> SOMETHING_TO_SIGN=41b27cb63e3be6074fd28cf5ee739151c92f2ef05f0a1a3cf5ae13de3007fc8e
> SIG=`eris-keys sign --addr $ADDR $SOMETHING_TO_SIGN`
> echo $SIG
7B96F6C19EA50BFF83DEA9C80616BDBDFC885C3E7321EAF92D212CE90B9EB5898FE87D95B0A8286E4A49D0F497223C2DAFD38D50E4F6F3A39F7F7B240FDCEC03
> eris-keys verify $SOMETHING_TO_SIGN $SIG $PUB
true
```

## Generate a key with a password

```
> eris-keys gen
Enter Password:****
5A87726028F91E1BC24DD051A3D7CABDBAC6DBD7
> ADDR=5A87726028F91E1BC24DD051A3D7CABDBAC6DBD7
> eris-keys sign --addr $ADDR $SOMETHING_TO_SIGN
account is locked
> eris-keys unlock --addr $ADDR
Enter Password:****
5A87726028F91E1BC24DD051A3D7CABDBAC6DBD7 unlocked
> eris-keys sign --addr $ADDR $SOMETHING_TO_SIGN
63C1563853EC12CB3EAF14EDC918AC2C5287943D0601434376D70C17380C674BB0EA9F1AC24EF3276D89AAED56E353F4AAD5B276BC3B0BB96EA0EB50EA95BA0F
```

Notice how the first time we try to sign, the account is locked. `eris-keys unlock` will unlock by default for 10 minutes. Use the `--time` flag to specify otherwise.

A key can be relocked with `eris-keys lock --addr $ADDR`

## Other key types

Use the `--type` flag to specify a key type. The tool currently supports:

- `secp256k1,sha3` (ethereum)
- `secp256k1,ripemd160sha256` (bitcoin)
- `ed25519,ripemd160` (tendermint)

The default is `ed25519,ripemd160`. The flag is only needed for `gen`, `import`, and `verify`.

## Names

Use a `--name` instead of the `--addr`:

```
> ADDR=`eris-keys gen --name mykey --no-pass`
> eris-keys pub --name mykey
4A976DD66E4245DC6BF06DA09A856C4E28CAA514CCAEC74976A47BCF1124801A
> eris-keys pub --addr $ADDR
4A976DD66E4245DC6BF06DA09A856C4E28CAA514CCAEC74976A47BCF1124801A
```

Use the `eris-keys name` command to change names, remove them, or list them.

## More 

Run `eris-keys` or `eris-keys <cmd> --help` for more.

# HTTP

Start the daemon with `eris-keys --host localhost --port 12345 server`

The endpoints:

### Generate keys
`/gen` 
	- Args: `auth`, `type`, `name`
	- Return:  newly generated address

### Manage keys
`/pub`
	- Args: `addr`, `name`
	- Return: the addresses' pubkey

`/sign`
	- Args: `msg`, `addr`, `name`
	- Return: the signature

`/unlock`
	- Args: `auth`, `addr`, `name`
	- Return: success statement

`/lock`
	- Args: `addr`, `name`
	- Return: success statement

`/import`
	- Args: `type`, `key`, `name`
	- Return: address

`/name`
	- Args: `rm`, `ls`, `name`, `addr`
	- Return: name, address, or list of names

### Utilities
`/verify`
	- Args: `addr`, `hash`, `sig`
	- Return: true or false

`/hash`
	- Args: `type` ("sha256", "ripemd160"), `data`
	- Return: hash value


All arguments are passed as a json encoded map in the body. The response is a struct with two strings: a return value and an error.

All arguments and return values that would be byte arrays are presumend hex encoded
