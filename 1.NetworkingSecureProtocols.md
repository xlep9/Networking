TLS (Transport Layer Security) is a cryptographic protocol that secures communication over networks by providing confidentiality and integrity.

HTTP

- Establish a TCP three-way handshake with the target server
- Communicate using the HTTP protocol; for example, issue HTTP requests, such as GET / HTTP/1.1

<img width="1100" height="696" alt="image" src="https://github.com/user-attachments/assets/9c5c0385-faa9-4cca-a759-6bf59ab54a81" />

HTTP Over TLS (HTTPS)

- Establish a TCP three-way handshake with the target server
- Establish a TLS session
- Communicate using the HTTP protocol; for example, issue HTTP requests, such as GET / HTTP/1.1

<img width="1060" height="800" alt="image" src="https://github.com/user-attachments/assets/fef36afd-0cbe-4fee-86f6-482ca52d0218" />

----------------------------

VPN

A VPN (Virtual Private Network) creates a secure tunnel between a client and a VPN server, allowing users to access remote networks as if they were directly connected. VPNs are widely used by companies to allow employees to securely access internal resources from remote locations.

Key Features:
- Encryption: VPNs encrypt all traffic between the client and server.
- Anonymity: VPN users appear to be coming from the VPN server’s IP address rather than their actual location.

------------------------------

TLS packet decrypted 

file "ssl-key.log" contains TLS keys

<img width="1100" height="767" alt="image" src="https://github.com/user-attachments/assets/824827d7-34ba-449d-990a-599efa820219" />

click the “Browse” button marked with four to locate the ssl-key.log

<img width="1100" height="816" alt="image" src="https://github.com/user-attachments/assets/8ddbb033-4f36-441f-a81b-92983259b368" />

Finally, click OK, and Wireshark will show all the TLS decrypted

<img width="1100" height="485" alt="image" src="https://github.com/user-attachments/assets/1efdad83-9d91-4640-872f-86fd74dc97fa" />

