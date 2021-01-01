# Security

Without a way to verify that a request has originated from who it is claimed to have,
anybody may send requests impersonating users (including administrators).

This propsal is based on a simplified version of the [HTTP Signatures](https://tools.ietf.org/id/draft-cavage-http-signatures-12.html)
protocol.

## Sending a Request

### Creating the `Digest` Header

Add the [`Digest`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Digest) header to every HTTP
request. For simplicity, this is restricted to `sha-512`.

#### How do I implement it?

1. `sha-512` hash the *body* of the HTTP request.
2. Base64 encode the output of the hash.
3. Add the header `Digest: sha-512=<base64_sha512_hash>` to the HTTP response, where `<base64_sha512_hash>`
   is the value from step 2.
   
##### Warning

Some libraries may give the hash output as hex (rather than binary). In such a case, when Base64
encoding, you must convert from hex to Base64, *not* binary to Base64.


### Creating the `Signature` Header

1. Make the public key available at `/fed/key`.
   * If your server does not support this security proposal, then the
     `/fed/key` endpoint should return a `501 Not Implemented` error.
   * The public key should be serialised using the ASN.1 structure
     `SubjectPublicKeyInfo` defined in X.509 and encoded using PEM.
2. Construct the following string based on the values from the HTTP request
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
3. The string from step 2 should be `rsa-sha512` signed using [PKCS #1](https://tools.ietf.org/html/rfc8017)
   (you will almost certainly need to use a library to do this).
4. The signature from step 3 should be Base64 encoded. See the
   [previous warning](#warning).
5. Send the following header in the HTTP request:
   ```
   Signature: keyId="rsa-global",algorithm="hs2019",headers="(request-target) host date digest",signature="<base64_signature>"
   ```
   All fields are static apart from the final signature field, which is the output from step 4.


## Receiving a Request

### Verifying the `Signature` Header

1. Obtain the public key from `http://<host>/fed/key`, where `<host>` is the value from the `Host` HTTP header.
   * If the server returns a `501` error, then it has not implemented this security proposal.
   * You may decide whether to accept the request without verification, or reject it.
   * You can not continue any further with verification (including verification of the
    `Digest` header).
2. Generate the string from step 1 of [Creating the Signature Header](#creating-the-signature-header).
3. Obtain the `<base64_signature>` from the `Signature` header:
```
Signature: keyId="global",algorithm="rsa-sha512",headers="(request-target) host date digest",signature="<base64_signature>"
```
4. Using the `rsa-sha512` algorithm, verify, using the public key from step 1, that `<base64_signature>` is
   valid using [PKCS #1](https://tools.ietf.org/html/rfc8017) (you will almost certainly need to use a library to do this).

### Verifying the `Digest` Header

1. `sha-512` hash the *body* of the HTTP request.
2. Base64 encode the output of the hash.
3. Verify that the output from step 2 matches the `<base64_sha512_hash>` value from the `Digest` header:
   `Digest: sha-512=<base64_sha512_hash>`.
   
## Some Examples

### Python

Uses the [crytography](https://cryptography.io/en/latest/index.html) library.

#### Exporting a Public Key

```python3
from cryptography.hazmat.primitives import serialization

with open("private.pem", "rb") as key_file:
   private_key = serialization.load_pem_private_key(
      key_file.read(),
      password=None,
   )   
   
public_key = private_key.public_key()
pem = public_key.public_bytes(
   encoding=serialization.Encoding.PEM,
   format=serialization.PublicFormat.SubjectPublicKeyInfo
)

print(pem.decode("utf-8"))
```

#### Generating a Digest

```python3
import base64
from cryptography.hazmat.primitives import hashes

digest = hashes.Hash(hashes.SHA512())
digest.update(b"A message")

base64encoded = base64.b64encode(digest.finalize())

print(base64encoded.decode("ascii"))
```

#### Signing a String

```python3
import base64
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding

with open("private.pem", "rb") as key_file:
   private_key = serialization.load_pem_private_key(
      key_file.read(),
      password=None,
   )   
  
message = b"A message I want to sign"
signature = private_key.sign(
    message,
    padding.PKCS1v15(),
    hashes.SHA512()
)

print(base64.b64encode(signature).decode("ascii"))
```

#### Verifying a Signature

```python3
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding

# You should get the public key from `/fed/key`.
with open("public.pem", "rb") as key_file:
    public_key = serialization.load_pem_public_key(key_file.read())

# Get the signature from the HTTP header.
encoded_signature = "ij3WqCSE+289DW4KV3Rh//4mZ1dev5I7m6rUrYyvcojmPBhuVzqxJ0XLoWPKtGz6aVYi4k+Ide1zTGUIAOWQEwCiT4WP/GrsYukwgAfgS9q80YgiKIyqVBvc953XLVzgnOT+8X2HQ/LTg+BwP23kLeEXPabxhMN323L+gVWVyoiIUYEf0B34PbPq/KTPqW/rHtup6ovSRfvy8Bqeqmtpmc0gJwR7WnKRYEiVn40yRQDxtO6zSjvmObv5U2BKCjprnOAp5yfKzROkpfqui1yjKMp5RfA+NILGiJSSQwgGe1eG0QOWYoW8JecLOrxBHOJMuFc0wDQ0k9cip/nAc/T5Cw=="

decoded_signature = base64.b64decode(encoded_signature)

# If this throws an InvalidSignature exception, then the signature
# was invalid.
public_key.verify(
    decoded_signature,
    message,
    padding.PKCS1v15(),
    hashes.SHA512()
)
```

### Node

#### Exporting a Public Key

```javascript
const crypto = require("crypto");
const fs = require("fs");

const publicKey = crypto.createPublicKey({
  key: fs.readFileSync("private.pem"),
  format: "pem"
});

const encodedPublicKey = publicKey.export({
  type: "spki",
  format: "pem"
});

console.log(encodedPublicKey);
```

#### Generating a Digest

```javascript
const crypto = require("crypto");

const hash = crypto.createHash("sha512");
hash.update("A message");

const digest = hash.digest("base64");

console.log(digest);
```

#### Signing a String

```javascript
const crypto = require("crypto");
const fs = require("fs");

const privateKey = crypto.createPrivateKey({
  key: fs.readFileSync("private.pem"),
  format: "pem"
});

const sign = crypto.createSign("SHA512");
sign.write("A message I want to sign");
sign.end();

const signature = sign.sign(privateKey, "base64");

console.log(signature);
```

#### Verifying a Signature

```javascript
const crypto = require("crypto");
const fs = require("fs");

const publicKey = crypto.createPublicKey({
  key: fs.readFileSync("private.pem"),
  format: "pem"
});

const signature = "ij3WqCSE+289DW4KV3Rh//4mZ1dev5I7m6rUrYyvcojmPBhuVzqxJ0XLoWPKtGz6aVYi4k+Ide1zTGUIAOWQEwCiT4WP/GrsYukwgAfgS9q80YgiKIyqVBvc953XLVzgnOT+8X2HQ/LTg+BwP23kLeEXPabxhMN323L+gVWVyoiIUYEf0B34PbPq/KTPqW/rHtup6ovSRfvy8Bqeqmtpmc0gJwR7WnKRYEiVn40yRQDxtO6zSjvmObv5U2BKCjprnOAp5yfKzROkpfqui1yjKMp5RfA+NILGiJSSQwgGe1eG0QOWYoW8JecLOrxBHOJMuFc0wDQ0k9cip/nAc/T5Cw==";

const verify = crypto.createVerify("SHA512");
verify.write("A message I want to sign");
verify.end();

const result = verify.verify(publicKey, signature, "base64");
console.log(result);
```
   
## N.B.

There has been a newly published (November 2020) diverging specification for signing HTTP messages:
[Signing HTTP Messages](https://www.ietf.org/archive/id/draft-ietf-httpbis-message-signatures-01.html).
This proposal is based on the predessor specification.

PKCS #1 is used over PSS to simplify implementation.

At the moment, a MITM attack could replace the public key at `/fed/key` with another. This can be solved
either by:
* enforcing HTTPS (perhaps using our own agreed upon certificate authority);
* using a X.509 certificiate instead of a public key, which we use our own authority to sign.
