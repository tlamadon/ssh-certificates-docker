
First setup the system with 

```
docker compose build
docker compose up
```

To run code on the different services (virtual computer), you can start a shell on each by running

```
docker compose exec ca sh
docker compose exec server sh
docker compose exec client sh
```

Before we get started we share the public key of the CA with the server and the client.

```sh
# we share the ca public key with the client and the server
ca> cp /etc/ssh/ca.pub /network-client
ca> cp /etc/ssh/ca.pub /network-server
```

## Client certificates

Using the CA signing, the server can accepts users authentification using the public key, without requiring copy that key first. Let's see how it works! 

```sh
# we add the CA as a Trusted source for keys on the servers and restart sshd
server> echo "TrustedUserCAKeys /network-server/ca.pub" >> /etc/ssh/sshd_config
server> kill -SIGHUP $(pgrep -f "sshd -D")

# we share the public key of user 1 with the CA
client> cp .ssh/user1.pub /network-client/user1.pub

# on the CA we create a certificate for user1
ca> ssh-keygen -s /etc/ssh/ca -V +52w -n user1 -I user1-key1 -z 1 /network-client/user1.pub

# back on the client, we take our newly created certificate and add it!
client> cp /network-client/user1-cert.pub ~/.ssh

# finally on the client, you can ssh using your user1 key
client> ssh -i .ssh/user1 server

# done! at this point we could login into the server without ever sharing the public key of the user with the server and without having to type in a password.

```

## Host certificates

Using the CA signing, the client can accept the host public key directly. Let's see how it works! 

```sh
# we share the public key of server with the CA
server> cp /etc/ssh/ssh_host_ecdsa_key.pub /network-server/

# on the CA we create a host certificate for server
ca> ssh-keygen -h -s /etc/ssh/ca -V '+10d' -I abcd -z 00001 -n server /network-server/ssh_host_ecdsa_key.pub  

# on the server, we make the certificate available in sshd
server> echo "HostCertificate /network-server/ssh_host_ecdsa_key-cert.pub" >> /etc/ssh/sshd_config
# we restart sshd
server> kill -SIGHUP $(pgrep -f "sshd -D")

# Back on the client, we trust the CA for "server"
client> echo "@cert-authority server $(cat /network-client/ca.pub)" >> ~/.ssh/known_hosts

# DONE, when doing ssh, we do need to accept a new public key from the server because it is signed by the CA!
```