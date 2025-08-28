# Index

- [Deploying Adguard](#deploying-adguard)
- [Tailscale](#tailscale)
- [Generating the CAs](#generating-the-cas)
    - [Root CA](#root-ca)
    - [Intermediate CA](#intermediate-ca)
    - [Signing](#signing)
- [Deploying Caddy](#deploying-caddy)
    - [Caddyfile](#caddyfile)
    - [Caddy](#caddy)

# Deploying Adguard

Edit the provided adguard-compose.yaml as needed for your use case.

In the Compose setup, I opted to use host network mode. While it would probably be more secure to use bridge mode and explicitly map the necessary ports, I chose host mode for most of the containers in this project because it simplified the initial setup. Now that I have a better understanding of Docker, I plan to experiment with bridge mode in the future.

Once Adguard Home is up and running, access it, and navigate to Filters > DNS Rewrites. Add a DNS Rewrite, write the wildcard, *.[your-domain], changing it to whatever you want your domain to be, and enter the IP of the device that will run your Caddy instance. In my case, it's the same machine I'm running Adguard on. 

# Tailscale

Tailscale is a zero-config VPN built on WireGuard. Unlike a normal VPN, it lets all your devices on your tailnet securely connect to each otheras if they’re on the same local network, without exposing anything to the public internet. Normally, Tailscale only routes traffic to the devices you add to your tailnet. However, subnet routing lets a device act like a gateway to an entire subnet, so other devices on your Tailnet can reach machines that aren’t running Tailscale directly. In my case, I already used my router as an exit node on my tailscale, so I also enabled it to advertise my Hosts VLAN, where my server resides. Whatever tailnet device you decide to use, access the command line and use:

sudo tailscale up --advertise-routes=192.168.50.0/24

Then navigate to the Tailscale Web Admin Console, Machines, the device you chose, and approve the route.

After that, navigate to the DNS tab, Add nameserver, insert your Adguard server from the route you're advertising, or the Tailscale IP it was provided, and tick Override DNS servers.

# Generating the CAs:

## Root CA

By default, Caddy’s tls internal command generates its own valid CAs and issues certificates automatically. However, the leaf certificates it creates are rotated every 7 days. I ended up running into problems with how Caddy generates its internal CAs, as I couldn’t get my sites to be fully trusted. To have more control over certificate lifetimes and management, I chose to create my own CA instead. From my own testing, in order for Firefox and devices in the Apple ecosystem to recognize a custom CA as valid, it must meet modern requirements: the certificates need to be signed with SHA-256 and have a maximum validity of 825 days.

Keep in mind that these commands are not for the most secure configuration:

openssl ecparam -name prime256v1 -genkey -noout -out root.key

nano root.cnf

```
[ req ]
distinguished_name = req
x509_extensions    = v3_ca
prompt             = no

[ req_distinguished_name ]
CN = [your-root-CA]

[ v3_ca ]
basicConstraints       = critical,CA:TRUE
keyUsage               = critical,keyCertSign,cRLSign
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer:always
```

openssl req -x509 -new -nodes -key root.key -sha256 -days 825 \
-config root.cnf -extensions v3_ca -out rootCA.crt

## Intermediate CA

openssl ecparam -name prime256v1 -genkey -noout -out intermediate.key

openssl req -new -key intermediate.key -out intermediate.csr -subj "/CN=Caddy Custom Intermediate CA"

nano intermediate.cnf

```
[ v3_intermediate_ca ]
basicConstraints       = critical,CA:TRUE,pathlen:0
keyUsage               = critical,keyCertSign,cRLSign
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer:always
```

openssl x509 -req -in intermediate.csr -CA rootCA.crt -CAkey root.key \
-CAcreateserial -out intermediate.crt -days 825 -sha256 \
-extfile intermediate.cnf -extensions v3_intermediate_ca

## Signing 

Firefox and Apple devices also require that certificates explicitly list all domains in the Subject Alternative Name (SAN) field. If a domain isn’t included there, the site won’t be trusted.

Here's what that would look like:

nano wildcard.cnf

```
[ req ] 
default_bits       = 2048 
prompt             = no 
default_md         = sha256 
distinguished_name = 
dn req_extensions  = req_ext

[ dn ] CN = [your-domain]

[ req_ext ] subjectAltName = @alt_names

[ alt_names ] 
DNS.1 = [your-domain] 
DNS.2 = \*.[your-domain]
DNS.3 = photos.[your-domain] 
DNS.4 = cloud.[your-domain]
DNS.5 = siem.[your-domain]
DNS.6 = adguard.[your-domain]
DNS.7 = vaultwarden.[your-domain]
DNS.8 = portainer.[your-domain]
DNS.9 = nas.[your-domain]
```

openssl req -new -key wildcard.key -out wildcard.csr -config wildcard.cnf

openssl x509 -req -in wildcard.csr -CA intermediate.crt -CAkey intermediate.key \
-CAcreateserial -out wildcard.crt -days 825 -sha256 -extfile wildcard.cnf -extensions req_ext

cat wildcard.crt intermediate.crt > wildcard-fullchain.crt

*.[your-domain] {
    tls /container/path/to/wildcard-fullchain.crt \
        /container/path/to/wildcard.key

# Deploying Caddy

## Caddyfile
Within my completed Caddyfile, you’ll see:
```
{
        transport http {
            tls_insecure_skip_verify
        }
    }
```
This is for services that generate their own certificates. It basically tells Caddy to pass the upstream certificate through without validation, so you don’t run into issues with both certificates conflicting. 

nano Caddyfile

```
*.[your-domain] {
    tls /container/path/to/wildcard-fullchain.crt \
        /container/path/to/wildcard.key
    @photos host photos.[your-domain]
    reverse_proxy @photos [device-ip]:2283

    @cloud host cloud.[your-domain]
    reverse_proxy @cloud [device-ip]:11000

    @siem host siem.[your-domain]
    reverse_proxy @siem [device-ip]:443 {
        transport http {
            tls_insecure_skip_verify
        }
    }

    @adguard host adguard.[your-domain]
    reverse_proxy @adguard [device-ip]:3000

    @vaultwarden host vaultwarden.[your-domain]
    reverse_proxy @vaultwarden [device-ip]:8000

    @nas host nas.[your-domain]
    reverse_proxy @nas [device-ip]:8006 {
        transport http {
            tls_insecure_skip_verify
        }
    }

    @unknown not host photos.[your-domain] cloud.[your-domain] siem.[your-domain] adguard.[your-domain] \
             vaultwarden.[your-domain] portainer.[your-domain] nas.[your-domain]
    respond @unknown "Unknown subdomain" 404
}
```
## Caddy

Once you've created your Caddyfile, spin up the caddy-compose.yaml, changing it as needed for your use case.
