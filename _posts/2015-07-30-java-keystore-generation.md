---
layout:     post
title:      "Java keystore generation regarding intermediate certificates"
date:       2015-07-30 08:38:00
categories: sysadmin
---

If you’re hosting a Java application server such as Tomcat or Jetty, you’d definitely like to configure SSL for the end-to-end security. Java (by default) implements its own SSL implementation, which instructs you to use keystore file to store private keys and certificates, using a special keytool command which comes installed with Java.

SSL Certificates signed by a trusted authority (such as RapidSSL) usually consist of some intermediate certificates, which needs to be served by the server to pass validity checks done by client’s browser. There are many guides on the web that fails in this part.

_Preliminary note: PEM format means a readable file, certificates start with `---BEGIN CERTIFICATE---` and private keys start with `-----BEGIN PRIVATE KEY-----` line._

Here’s an easy to follow guide. Start with an empty directory.  
Skip to Step 2 if you have private key (PEM encoded .key)  
Skip to Step 3 if you have certificate signing request (PEM encoded .csr)  
Skip to Step 4 if you have your certificate (PEM encoded .crt or .pem)  

**STEP 1.** Prepare (password-less) private key.

    openssl genrsa -des3 -passout pass:1 -out domain.pass.key 2048
    openssl rsa -passin pass:1 -in domain.pass.key -out domain.key
    rm domain.pass.key

**STEP 2.** Prepare certificate signing request (CSR). We’ll generate this using our key. Enter relevant information when asked. Note the use of `-sha256`, without it, modern browsers will generate a warning.

    openssl req -key domain.key -sha256 -new -out domain.csr

**STEP 3.** Prepare certificate. Pick one:

**a)** Sign it yourself.

    openssl x509 -req -days 3650 -in domain.csr -signkey domain.key -out domain.crt

**b)** Send it to an authority. Your SSL provider will supply you with your certificate and their intermediate certificates in PEM format.

**STEP 4.** Add to trust chain and package it in PKCS12 format. First command sets a keystore password for convenience (else you’ll need to enter password a dozen times). Set a different password for safety.

    export PASS=LW33Lk714l9l8Iv

Pick one:

**a)** Self-signed certificate (no need for intermediate certificates)

    openssl pkcs12 -export -in domain.crt -inkey domain.key -out domain.p12 -name domain -passout pass:$PASS

**b)** Need to include intermediate certificates

Download intermediate certificates and concat them into one file. The order should be sub to root.

    cat sub.class1.server.ca.pem ca.pem > ca_chain.pem

Use a -caname parameter for each intermediate certificate in chain file, respective to the order they were put into the chain file.

    openssl pkcs12 -export -in domain.crt -inkey domain.key \
    -out domain.p12 -name domain -passout pass:$PASS \
    -CAfile ca_chain.pem -caname sub1 -caname root -chain

**STEP 5.** Create keystore. This command imports .p12 file from previous step. This is required as we’re not generating the private key using keytool, because you can reuse a key assuming it’s not compromised.

    keytool -importkeystore -deststorepass $PASS -destkeypass $PASS \
    -destkeystore domain.keystore -srckeystore domain.p12 -srcstoretype PKCS12 \
    -srcstorepass $PASS -alias domain

_Important note_: Although `keytool -list` will only list one entry and not any intermediate certificates, it will work perfectly as all intermediate certificates would have been merged into that entry.

**STEP 6.** Use the generated `domain.keystore` file in your Java application container. Use an [SSL checker service](https://certlogik.com/ssl-checker/) to verify your intermediate certificates are OK.
