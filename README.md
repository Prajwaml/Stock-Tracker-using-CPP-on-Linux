# Stock-Tracker-using-CPP-on-Linux
This project demonstrates a **multi-user stock tracker** application secured with **mutual TLS (mTLS)**.   Clients must present a valid certificate signed by the CA in order to connect to the server.

## Prerequisites
- Ubuntu or Linux (tested on Ubuntu 22.04 with VMware)
- `openssl`
- `nginx` (used as reverse proxy with mTLS enabled)
- `curl`
- `python3` (optional, for testing clients with Python)

- ## Certificate Setup

### 1. Generate a Certificate Authority (CA)
mkdir certs && cd certs

# Generate CA private key
openssl genrsa -out ca.key 4096

# Self-sign CA certificate
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt

### 2. Generate Server Certificate
# Server key
openssl genrsa -out server.key 2048

# CSR
openssl req -new -key server.key -out server.csr

# Sign with CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out server.crt -days 365 -sha256

### 3. Generate Client Certificates
# Client 1
openssl genrsa -out client1.key 2048
openssl req -new -key client1.key -out client1.csr
openssl x509 -req -in client1.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out client1.crt -days 365

# Client 2
openssl genrsa -out client2.key 2048
openssl req -new -key client2.key -out client2.csr
openssl x509 -req -in client2.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out client2.crt -days 365

certs/
├── ca.crt
├── ca.key
├── server.crt
├── server.key
├── client1.crt
├── client1.key
├── client2.crt
├── client2.key


<----------Running the Application---------->
1. Start the Stock Tracker Backend
   cd ~/stock-tracker/build
   ./stock-tracker --port 8080

2. Configure NGINX for mTLS
   Edit /etc/nginx/sites-enabled/default:
     server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /home/prajwal/stock-tracker/certs/server.crt;
    ssl_certificate_key /home/prajwal/stock-tracker/certs/server.key;
    ssl_client_certificate /home/prajwal/stock-tracker/certs/ca.crt;
    ssl_verify_client on;

    location / {
        proxy_pass http://127.0.0.1:8080;
      }
    }
    Reload NGINX:
      sudo nginx -s reload

<----------Testing mTLS---------->
##Using curl
Client 1
curl --cert certs/client1.crt --key certs/client1.key --cacert certs/ca.crt https://localhost

Client 2
curl --cert certs/client2.crt --key certs/client2.key --cacert certs/ca.crt https://localhost

If you try without a client cert:
curl --cacert certs/ca.crt https://localhost

You should see:
400 No required SSL certificate was sent

Using Python Requests:
import requests

url = "https://localhost"

# Client 1
resp1 = requests.get(
    url,
    cert=("certs/client1.crt", "certs/client1.key"),
    verify="certs/ca.crt"
)
print("Client1:", resp1.text)

# Client 2
resp2 = requests.get(
    url,
    cert=("certs/client2.crt", "certs/client2.key"),
    verify="certs/ca.crt"
)
print("Client2:", resp2.text)
