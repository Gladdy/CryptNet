# End-to-end encrypted datagram service

Features:
* low network overhead for the security offered
    ~10 bytes for dns
    ~10 bytes for every group member
    ~250bytes for an encrypted public key for broadcasting (if necessary)
* supports messages, files and streams
* to multiple recipients
* anonymous data content (end-to-end encrypted)
* anonymous senders
* only the end users know the group compositions

Problems:
* sender bandwidth increases with group size (fixed by adding 1 or 2 additional broadcast proxies depending on the load situation)
* keeping participants anonymous whilst verifying their authenticity (solved: use a passphrase for adding people)
* sending data to anonymous receivers (no need for anonymous receivers due unknown message contents and anonymous transmitters)
* The proxies (especially L2) should not know CS_private
* Latency due to a triple proxy (can be shortened to a double proxy during high traffic, getting rid of L3, masking patterns)
* If you have control over all L23 and CS you can trace the path of the packet.

#### Setup:
All devices keep a connection with the central server for receiving messages.
All non-mobile (not network limited) devices not behind a NAT (ie. company servers) act as proxies for sending traffic.
Get data from the central server: its public key and a list of proxies with their public keys.

#### Creating a group:
* generate a key pair
* add yourself to the recipient list

#### Adding a person:
##### The adder
* prompt for a target id
* prompt for a passphrase
* encode the group key pair and recipient list with the passphrase
* send the message to the target id

##### The addee
* if their private keys don't work
* prompt for a passphrase
* if it works, add the recipient list and private key to its cache
* for every recipient except itself
    generate a temporary key       -- hide similar looking messages for the central server
    encode the recipient list with the temporary key
    prepend the temporary key
    encrypt the entire string with the groups public key and send

### Sending a message
0. encrypt with group_pub (or passphrase) and append the encrypted (with CS_pub) recipients
1. pick a proxy at random (L2)
2. S sends L2 the group_pub encrypted message with (1) encrypt(L2_pub,group_pub) (2) a list of targets encrypted with CS_pub
<!-- 3. L1 sends the message to the proper L2 (stripping it of which L2 to use) -->
4. L2 decodes the group public key and generates a temp key.
5. L2 scrambles the message using the temp key (encode and prepend the key)
6. L2 encrypts the scrambled message with group_pub
7. L2 sends individual messages for the different targets with the (now different) 'encrypted_scrambled_encrypted_message's to L3
8. L3 proxy forwards the message to CS
9. CS decrypts the target of the message and sends it there
10.T applies group_private, retrieves and decrypts the scrambling and finally decrypts the message with group_private to get cleartext.


### Layers:
S   : encrypting and sending the message with encrypt(L2_pub, group_pub) for scrambling, a choice for L2 and a list of encrypt(CS_pub, target)
      knows everything about the message

<!--
L1  : hiding the source of the message for the rest of the chain, simply forwarding
      knows the source of the message, the choice of L2, encrypt(L2_pub, group_pub) and all encrypted targets
      does not know the target -->

L2  : splices the message into separate, scrambled parts (is allowed to know group_pub because it doesn't know the source or destinations)
      knows the L1 it came from, knows the L3 it moves toward, knows group_pub, knows the encrypted_scrambled_encrypted_message, knows all encrypted targets
      does not know the target

L3  : hiding that messages are part of the same group and forwarding them to CS (can be omitted if the outgoing link of L2 is congested with random interleaving)
      knows the L2 the message came from, knows the encrypted target, knows the encrypted_scrambled_encrypted_message
      does not know the source or target

CS  : sending the messages to their final destinations
      knows the choice of L3, knows the target address, knows the encrypted_scrambled_encrypted_message
      does not know the source

T   : receives the message (target)
      knows the content of the message
      doesn't know the source or the other recipients

