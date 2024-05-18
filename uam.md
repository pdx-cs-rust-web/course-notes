# Web Security; User Authentication and Management

## Security Vocabulary

* Threat Model: expected attacks and response
* Arbitrary Code Execution: unintentionally running external code
* Escalation of Privilege: getting more privs than the source has
* SQL Injection: messing with the database
* Denial of Service: making the server unusable /
  unavailable
* Reflection: using servers for Distributed DOS attacks on others
* Firewall: network filter against unexpected threats
* Defense In Depth: layer protections
* User Data Exposure: privacy violations
* Monster-in-the-middle Attack: modifying traffic going both ways
* Traffic Analysis: watching encrypted traffic

## Rust?

* Types and borrow checking and memory safety helps
* Does not protect against arbitrary bugs
* Efficiency helps against DOS

## TLS (SSL)

* Encrypts packets over the wire
* "Symmetric-key" cipher
  * encryption: "plaintext" under "key" gives "ciphertext"
  * decryption: "ciphertext" under "key" gives "plaintext"
* "Public-key" cipher
  * encryption: "plaintext" under "public key" gives "ciphertext"
  * decryption: "ciphertext" under "private key" gives "plaintext"
  * vulnerable to MitM attacks with key spoofing
* Public Key Infrastructure, Certificate Authority
* Use "Let's Encrypt" CA via `certbot`
* Build TLS into your service *vs* let Apache / whatever
  deal with it (do this second thing)
