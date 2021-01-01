# Security

Without a way to verify that a request has originated from who it is claimed to have,
anybody may send requests impersonating users (including administrators).

This propsal is based on a simplified version of the [HTTP Signatures](https://tools.ietf.org/html/draft-cavage-http-signatures-08)
protocol.

## Servers

### Creating the `Digest` Header

Add the [`Digest`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Digest) header to every HTTP
response. For simplicity, this is restricted to `sha-512`.

#### How do I implement it?

1. `sha-512` hash the *body* of the HTTP request.
2. Base64 encode the output of the hash.
3. Add the header `Digest: sha-512=<base64_sha512_hash>` to the HTTP response, where `<base64_sha512_hash>`
   is the value from step 2.
   
##### Warning

Some libraries may give the hash output as hex (rather than binary). In such a case, when Base64
encoding, you must convert from hex to Base64, *not* plain text to Base64.


### Creating the `Signature` Header

#### Step 1
Construct the following string based on the values from the HTTP response
(note that there is a line ending `\n` on all lines apart from the last):

```
(request-target): post /fed/posts
host: cooldomain.edu:8080
date: Tue, 07 Jun 2021 20:51:35 GMT
digest: SHA-512=X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=
```

There is a single space after each colon `:`. The part before the colon must be all lower-case. The part
following the colon (after the space) should be taken verbatim from the source (do not change the case).
The order of the key/value pairs MUST be the same as above.

* `(request-target)` - `<HTTP Method> <Request Path>` (separated by a single space).
  * `<HTTP Method>` - `GET`, `POST`, `PUT` or `DELETE`, made lower-case.
  * `<Request Path>` e.g. `/fed/posts`.
* `host` - The value from the `Host` HTTP header.
* `date` - The value from the `Date` HTTP header.
* `digest` - The value from the `Digest` HTTP header.

#### Step 2
The string from step 1 should be `rsa-sha512` signed.

#### Step 3
The signature from step 2 should be Base64 encoded. See the [previous warning](#warning).

#### Step 4
Send the following header in the HTTP response:
```
Signature: keyId="global",algorithm="rsa-sha512",headers="(request-target) host date digest",signature="<base64_signature>"
```
All fields are static apart from the final signature field, which is the output from step 4.

#### Step 5
Make the public key available at `/fed/key`.


## Clients

### Verifying the `Digest` Header

1. `sha-512` hash the *body* of the HTTP request.
2. Base64 encode the output of the hash.
3. Verify that the output from step 2 matches the `<base64_sha512_hash>` value from the `Digest` header:
   `Digest: sha-512=<base64_sha512_hash>`.
   
### Verifying the `Signature` Header

1. Generate the string from step 1 of [Creating the Signature Header](#creating-the-signature-header).
2. Obtain the public key from `http://<host>/fed/key`, where `<host>` is the value from the `Host` HTTP header.
3. Obtain the `<base64_signature>` from the `Signature` header:
```
Signature: keyId="global",algorithm="rsa-sha512",headers="(request-target) host date digest",signature="<base64_signature>"
```
4. Using the `rsa-sha512` algorithm, verify, using the public key from step 2, that `<base64_signature>` is
   valid.
