+++
title = 'Eclipse Mosquitto: Leveraging Certificates for Encryption, Identification, and ACLs'
date = 2024-01-16T06:50:50Z
description = "Explore the integration of TLS certificates with Mosquitto for enhanced IoT security. This post covers the generation of CA, server, and client certificates, configuring Mosquitto for encrypted communication and client authentication, and setting up ACLs for precise access control based on client identity. Ideal for IoT developers and security enthusiasts looking to deepen their understanding of MQTT security practices."
tags = ["MQTT", "Mosquitto", "TLS Certificates", "IoT Security", "Access Control Lists", "Client Authentication", "SSL/TLS", "MQTT Broker", "IoT Communication"]
categories = ["IoT Security"]
draft = false
+++


This blog post delves into the sophisticated use of TLS certificates with Mosquitto, transcending their conventional role as encryption tools to serve as pivotal elements in client authentication and authorization.

We’ll explore how certificates, typically recognized for their role in securing communications, can be ingeniously repurposed for client identification through their Common Name (CN) attribute. This approach not only enhances the security posture of your MQTT ecosystem but also simplifies the authentication process. Furthermore, we will discuss how Access Control Lists (ACLs) can be seamlessly integrated with this identification mechanism, enabling precise control over who can publish or subscribe to specific topics. This synergy of encryption, identification, and access control not only fortifies the security of MQTT communications but also paves the way for more streamlined and manageable client permissions.

To implement the security measures discussed, we need to configure Mosquitto appropriately. Let's break down the essential elements of the configuration:

```plaintext
# This line tells Mosquitto to listen for MQTT connections on port 8883
# (the standard port for MQTT over TLS) and to accept connections from any IP address.
listener 8883 0.0.0.0

# These specify the paths to the CA certificate, the server's certificate,
# and the server's private key, respectively.
cafile /mosquitto/certs/ca.pem
certfile /mosquitto/certs/server.pem
keyfile /mosquitto/certs/server.key

# By setting require_certificate to true, a client connecting to this listener must
# provide a valid certificate in order for the network connection to proceed.
# This allows access to the broker to be controlled outside of the mechanisms provided by MQTT.
require_certificate true

# If require_certificate is true, you may set use_identity_as_username to true
# to use the CN value from the client certificate as a username.
use_identity_as_username true

# This points to the Access Control List (ACL) file, which contains rules defining the
# permissions for different users, based on their usernames - in our case,
# the identities derived from their certificates.
acl_file /mosquitto/acl/acl.acl
```

## Generating Certificates

To secure our Mosquitto setup, we'll generate our own root Certificate Authority (CA), server certificates, and client certificates.

**Note**: Use PEM format and `.pem` file extension, to avoid certificate validation issues.

Here's how this is done:

### Root Certificate Authority Generation

1. **Generate CA Private Key**:
This command generates a 2048-bit RSA private key for our CA and saves it to ca.key.

```bash
openssl genrsa -out ca.key 2048
```

2. **Create CA Certificate**:
Here, we're creating a self-signed root CA certificate using the previously generated private key. The `-x509` flag indicates this is a self-signed certificate, `-sha256` specifies the use of SHA-256 hashing algorithm, and `-days` 1024 sets the certificate to be valid for 1024 days.

```bash
openssl req -x509 -new -nodes -key ca.key -sha256 -days 1024 -out ca.pem
```

### Server Certificates Generation

1. **Generate Server Private Key**:
This generates the server's 2048-bit RSA private key.

```bash
openssl genrsa -out server.key 2048
```

2. **Create a Certificate Signing Request (CSR) for the Server**:
The CSR contains information like the organization's name and the server's domain name (Common Name). As a good practice,
provide the domain name or a logical name to the "Common Name" section.

```bash
openssl req -new -key server.key -out server.csr
```

3. **Sign the Server CSR with the CA**:
This command uses the root CA to sign the server's CSR, generating the `server.pem` certificate valid for 500 days.
```bash
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out server.pem -days 500 -sha256
```

## Client Certificates Generation

In scenarios where we have multiple devices communicating via MQTT, it's crucial to enforce fine-grained access control. A common approach is to generate unique client certificates for each device, using the device ID as part of the certificate's Common Name (CN), combined with the server configuration `use_identity_as_username true` (see above). This setup allows us to restrict each device to specific topics in the Mosquitto broker using ACLs.

1. **Generate Device Private Key**:
```bash
openssl genrsa -out Device123.key 2048
```

2. **Create a CSR for the Device**:
When prompted for the 'Common Name', use the device's unique identifier (e.g., Device123).
```bash
openssl req -new -key Device123.key -out Device123.csr
```

3. **Sign the Device CSR with the CA**:
Repeat these steps for each device, replacing Device123 with the respective device ID.

```bash
openssl x509 -req -in Device123.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out Device123.pem -days 500 -sha256
```

## Configuring ACL for Device-Specific Topic Access

With the client certificates in place, we can now configure Mosquitto's ACL file to restrict access based on the device ID. Here's how to set up the ACL:

1. **Edit the ACL File (/mosquitto/acl/acl.acl)**:
In this configuration, `%u` will be replaced by the username of the connecting client (i.e., the device ID as per the client certificate's CN). For instance, if a device with the CN `Device123` connects, `%u` will be replaced with `Device123`, effectively allowing the device to access topics under `/device/Device123/#`.

```plaintext
user Device123
topic readwrite /device/%u/#
```

By leveraging dynamic placeholders like `%u` in our ACL file, we achieve a flexible yet secure way of managing topic access across a multitude of devices. This method not only reinforces the security of our MQTT communications but also introduces scalability and ease of management in complex IoT environments
By using the device's ID in both the certificate and the ACL configuration, we create a secure and straightforward way to ensure that each device can only access its designated topics. This method enhances the security and integrity of our MQTT communication.

## Final Steps: Verifying Your Secure MQTT Setup

After configuring Mosquitto and generating the necessary certificates, it's important to verify that the setup works as expected. Let's go through a couple of verification steps to ensure that our MQTT environment is secure and functional.

### Verifying TLS Communication with OpenSSL

To test the TLS handshake and verify that the server is correctly using the provided certificates, we can use the `openssl s_client` command:

```bash
openssl s_client -connect <server-address>:8883 -CAfile ca.pem -cert Device123.pem -key Device123.key
```

Look for messages indicating a successful TLS handshake. This will confirm that the client can establish a secure connection with the MQTT broker using the provided certificates.

## Testing MQTT Connection with Mosquitto Client

Finally, let's test the MQTT publish functionality using `mosquitto_pub`:

```bash
mosquitto_pub -h <server-address> -p 8883 --cafile ca.pem --cert Device123.pem --key Device123.key -t /device/Device123/ -m "22.5" --insecure -d
```

Successful execution of this command will publish the message "22.5" to the specified topic, demonstrating that your MQTT client can securely communicate with the broker using TLS.


## Wrapping Up
By following the steps in this guide, you've set
up a secure MQTT environment using Mosquitto, where communication is encrypted, and clients are authenticated and authorized using TLS certificates. This approach not only enhances the security of your MQTT communication but also provides a scalable way to manage client access through ACLs.

Remember, regular maintenance and updates to your Mosquitto setup and certificates are essential to keep your IoT environment secure.
