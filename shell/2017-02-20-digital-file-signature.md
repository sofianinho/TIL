# Digitally sign and verify any file

## Do you already have your certificate ?
### Generate a private key and an associated certificate
```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/selfsigned.key -out /tmp/selfsigned.crt
```
For the certificate, you have to fill in certain information.
### Generate strong ![Diffie-Hellman group](https://en.wikipedia.org/wiki/Forward_secrecy)
```sh
openssl dhparam -out /tmp/dhparam.pem 2048
```

## Digital signature before sending the file
In this step, you want to digitally sign `file.txt` using the previously generated keys.
```sh
openssl dgst -sha256 -sign /tmp/selfsigned.key -out signature file.txt
```
The signature of `file.txt` is in `signature`. The sender needs to transmit both.

## Verify the signature once the file and the signature received
1. Generate the public key from the public certificate used to sign
 ```sh
 openssl x509 -in selfsigned.crt -pubkey -noout > selfsigned.pub
 ```
2. The actual signature verification
 ```sh
 openssl dgst -sha256  -verify selfsigned.pub -signature signature file.txt
 ```
The correct output is `Verified OK` and incorrect signature output is `Verification Failure`

