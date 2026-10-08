# Project 10 — AWS Nginx Load Balancer with SSL/TLS

## Project Overview

I built a web-serving architecture on AWS using three EC2 instances: two Ubuntu backend servers running Apache and a separate Ubuntu server running Nginx as a reverse proxy and load balancer.

I began by checking the backend services and their individual responses. I then configured Nginx to distribute requests between them, tested the behaviour when one backend was unavailable, and associated an Elastic IP with the load balancer. Finally, I connected a domain name, configured HTTPS using Let’s Encrypt and Certbot, and tested HTTP-to-HTTPS redirection and certificate renewal.

The project helped me understand more than the installation commands. I worked through the complete request path, learned how to isolate failures at different layers, and kept evidence of the errors I encountered alongside the successful tests.

This repository documents that implementation through configuration examples, explanations, and screenshots.

## Project at a Glance

| Area | Project implementation |
|---|---|
| Platform | Amazon Web Services |
| Infrastructure | Three EC2 instances |
| Operating system | Ubuntu Linux |
| Backend services | Apache HTTP Server on two instances |
| Public-facing service | Nginx reverse proxy and load balancer |
| Backend naming | Local hostname resolution through `/etc/hosts` |
| Traffic distribution | Round-robin upstream configuration |
| Public address | AWS Elastic IP associated with the Nginx instance |
| Domain access | Public DNS pointing to the load balancer |
| HTTPS | Let's Encrypt certificate managed through Certbot |
| TLS design | TLS terminates at Nginx; backend requests use HTTP |
| Validation | Service checks, curl requests, browser tests, and configuration checks |
| Resilience exercise | Testing with one backend unavailable |
| Certificate lifecycle | Certbot renewal dry-run test |

## Project Objectives

A single web server provides a straightforward starting point, but it leaves all requests dependent on one backend. This project introduced a separate entry point that could distribute requests across two servers.

I wanted to understand how the load balancer, web services, network addresses, domain name, and certificate worked together. My objectives were to:

- Verify both Apache servers independently before configuring the proxy.
- Connect the load balancer to the backends through their private addresses.
- Make backend selection observable using different response identifiers.
- Configure Nginx to distribute requests across the upstream pool.
- Investigate behaviour when one backend became unavailable.
- Provide domain-based access through an Elastic IP.
- Enable HTTPS and redirect HTTP requests to the secure endpoint.
- Test the certificate-renewal workflow.
- Document troubleshooting clearly enough that another engineer could follow my reasoning.

## Architecture

```mermaid
flowchart TD
    client["Client: browser or curl"]
    dns["Public DNS: domain lookup"]
    eip["Elastic IP: public entry point"]
    lb["EC2: Nginx load balancer"]
    web1["EC2: AppNode1 / Web1"]
    web2["EC2: AppNode2 / Web2"]

    client -.->|"Resolve domain"| dns
    dns -.->|"Returns Elastic IP"| client
    client -->|"HTTP 80 or HTTPS 443"| eip
    eip --> lb
    lb -->|"Private network: HTTP 80"| web1
    lb -->|"Private network: HTTP 80"| web2
```

DNS resolves the domain name; it does not forward application traffic. Once the client has the public address, it connects to the load balancer.

### Server Responsibilities

| Server | Role |
|---|---|
| Nginx load balancer | Receives client traffic, terminates TLS, selects an upstream, and proxies requests |
| AppNode1 / Web1 | Runs Apache and serves the first identifiable backend response |
| AppNode2 / Web2 | Runs Apache and serves the second identifiable backend response |

The manual uses `Web1` and `Web2` as upstream names. My screenshot filenames use `AppNode1` and `AppNode2` for the backend servers. These refer to the same two backend roles in this documentation.

### How a Request Travels Through the System

1. The client resolves the domain through DNS.
2. DNS returns the load balancer's Elastic IP.
3. The client connects to the public endpoint.
4. For HTTPS, the client establishes a TLS connection with Nginx.
5. Nginx processes the request and selects a backend from the upstream pool.
6. Nginx sends the request to that backend over HTTP.
7. Apache returns its response to Nginx.
8. Nginx returns the response to the client over the client connection.

This separates the public connection from the backend connection. The two connections can use different protocols.

### TLS Termination

In this design, the encrypted connection ends at Nginx. The connection from Nginx to each Apache server uses HTTP over the private network.

HTTPS therefore protects the client-to-Nginx connection. It does not imply that the backend connections are encrypted. Backend TLS would be an additional improvement.

## Tools, Software, and Resources

| Tool or resource | Purpose |
|---|---|
| AWS account and Management Console | Creating and managing the cloud resources |
| Amazon EC2 | Hosting the three servers |
| AWS VPC networking | Providing the network environment for the instances |
| AWS Security Groups | Controlling permitted traffic |
| AWS Elastic IP | Providing a stable public IPv4 address |
| Ubuntu Linux | Running the web and proxy services |
| Apache HTTP Server | Serving backend content |
| Nginx | Reverse proxying, load balancing, and TLS termination |
| SSH and an EC2 key pair | Accessing the servers for administration |
| Bash | Running Linux administration and validation commands |
| `/etc/hosts` | Mapping backend names to private addresses on the load balancer |
| systemctl | Managing and inspecting services |
| curl | Testing backend and public responses |
| Domain name and DNS settings | Providing a named public endpoint |
| Snap / snapd | Supporting the Certbot installation |
| Certbot | Requesting, deploying, and testing certificate renewal |
| Let's Encrypt | Issuing the publicly trusted certificate |
| Web browser | Checking the public IP and domain responses |
| Visual Studio Code | Editing documentation and organizing evidence |
| Git and GitHub | Tracking and publishing the project |
| Local laptop | Providing the terminal, browser, editor, and screenshot workflow |


## Repository Structure

This repository currently uses a root-level README and a root-level screenshots folder.

```mermaid
flowchart TD
    repo["Project repository"]
    readme["README.md"]
    shots["screenshots/"]
    infrastructure["Infrastructure and backend evidence"]
    proxy["Nginx and load-balancing evidence"]
    public["Elastic IP and domain evidence"]
    tls["HTTPS and renewal evidence"]
    troubleshooting["Error and recovery evidence"]

    repo --> readme
    repo --> shots
    shots --> infrastructure
    shots --> proxy
    shots --> public
    shots --> tls
    shots --> troubleshooting
```


## Implementation Walkthrough

### Phase 1 — Establishing the Infrastructure

I prepared the three-instance environment so that the backend services and the public-facing proxy had separate roles.

This separation made testing easier. I could first check Apache on each server, then investigate the network path from Nginx, and finally test the public endpoint.

**EC2 instance evidence**

![Three EC2 instances running](screenshots/01-ec2-instances-running.png)

### Phase 2 — Preparing the First Apache Backend

I accessed the first Ubuntu web server through SSH and checked the Apache setup.

The first checkpoint was the backend service itself. Before introducing Nginx, I needed to establish that Apache could serve a response on its own server.

**SSH access to the first backend**

![SSH access to the first Ubuntu backend](screenshots/02-web1-ssh-ubuntu-login.png)

**Apache enabled on AppNode1**

![Apache enabled on AppNode1](screenshots/03-appnode1-apache2-enabled.png)

### Phase 3 — Preparing the Second Apache Backend

I repeated the backend preparation on AppNode2. Both servers needed to be usable independently before they could form a meaningful upstream pool.

**SSH access to the second backend**

![SSH access to AppNode2](screenshots/04-appnode2-ssh-ubuntu-login.png)

**Apache enabled on AppNode2**

![Apache enabled on AppNode2](screenshots/05-appnode2-apache2-enabled.png)

### Phase 4 — Making Backend Selection Visible

Identical pages would make it difficult to tell which server handled a request. I used distinguishable backend responses so that the load-balancing tests could identify the responding server.

The manual's approach uses a separate file, `lb-test.txt`, rather than replacing the application's homepage.

Example setup on Web1:

```bash
echo "Request handled by WEB1" | sudo tee /var/www/html/lb-test.txt
```

Example setup on Web2:

```bash
echo "Request handled by WEB2" | sudo tee /var/www/html/lb-test.txt
```

The important feature is that the two responses differ.

**Backend identifier evidence**

![Distinguishable backend responses](screenshots/06-backend-identifiers.png)

**Local response check on AppNode2**

![Apache response on AppNode2](screenshots/07-confiremed-curl-localhots-apache-on-appnode2.png)

**Response check on AppNode1**

![Apache response test on AppNode1](screenshots/08-confirmed-apache-test-onappnode1.png)

### Phase 5 — Accessing the Load Balancer and Configuring Backend Names

I accessed the Ubuntu load-balancer instance and configured local name resolution for the backend servers.

The `/etc/hosts` entries belong on the Nginx server. They allow that machine to resolve the backend names to their intended addresses; they do not create public DNS records.

The configuration pattern is:

```text
WEB1_PRIVATE_IP Web1
WEB2_PRIVATE_IP Web2
```

The uppercase address placeholders must be replaced with the actual private IPv4 addresses.

Useful checks at this stage are:

```bash
getent hosts Web1
getent hosts Web2

curl http://Web1/lb-test.txt
curl http://Web2/lb-test.txt
```

The key checkpoint is that both backends respond directly from the load balancer before the proxy configuration is tested.

**SSH access to the load balancer**

![Ubuntu load-balancer SSH access](screenshots/08-loadbalancer-ubuntu-login.png)

**Hosts-file configuration**

![Backend mappings in the load-balancer hosts file](screenshots/09-lb-host-contents.png)

### Phase 6 — Configuring Nginx

Nginx performs two related jobs in this project. As a reverse proxy, it receives requests on behalf of the Apache servers. As a load balancer, it selects which backend receives each request.

The following is the HTTP configuration pattern from the project manual. It describes the intended upstream behaviour before Certbot adds the HTTPS configuration.

It is a reference configuration, not a verbatim export of the final server file.

```nginx
upstream tooling_backend {
    server Web1:80 weight=5 max_fails=3 fail_timeout=30s;
    server Web2:80 weight=5 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    listen [::]:80;

    server_name YOUR_DOMAIN www.YOUR_DOMAIN;

    location / {
        proxy_pass http://tooling_backend;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Understanding the Main Directives

| Directive | Purpose |
|---|---|
| `upstream tooling_backend` | Defines the backend server pool |
| `server Web1:80` / `server Web2:80` | Identifies each upstream HTTP endpoint |
| `weight=5` on both servers | Gives the two servers equal relative weighting |
| `max_fails=3 fail_timeout=30s` | Configures passive failure accounting and temporary unavailability behaviour |
| `proxy_pass` | Forwards requests to the upstream pool |
| `Host` | Passes the request host to the backend |
| `X-Real-IP` | Provides the client address seen by Nginx |
| `X-Forwarded-For` | Appends the client address to the forwarded-address chain |
| `X-Forwarded-Proto` | Communicates whether the client used HTTP or HTTPS |

With no alternative algorithm selected, Nginx uses round robin. Equal weights support approximately equal distribution across available backends over suitable request samples; they do not guarantee a particular order for every request under every condition.

The failure settings are passive. Nginx learns about failures through actual upstream interactions rather than continuously probing the servers through this configuration.

### Phase 7 — Checking Configuration, Services, and Connectivity

I checked configuration validity and service state separately from request handling.

My validation sequence focused on three questions:

1. Can Nginx parse the configuration?
2. Are the required services running?
3. Can the load balancer obtain a response from each backend?

A typical configuration-change workflow is:

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx
```

The reload should only follow a successful configuration check.

**Initial Nginx syntax validation**

![Nginx configuration syntax validation](screenshots/10-nginx-config-syntax-ok.png)

**AppNode1 service state**

![Apache running on AppNode1](screenshots/11-apache-running-on-appnode1.png)

**AppNode1 response from the load balancer**

![AppNode1 responding to the load balancer](screenshots/12-backend-appnode1-respons-to-lb.png)

**AppNode2 service state**

![Apache running on AppNode2](screenshots/13-apache-running-on-appnode2.png)

**AppNode2 response from the load balancer**

![AppNode2 responding to the load balancer](screenshots/14-backend-appnode2-respons-to-lb.png)

**Nginx service state**

![Nginx running on the load balancer](screenshots/15-nginx-running-on-loadbalancer.png)

**Successful configuration validation after troubleshooting**

![Successful Nginx syntax check after resolving the configuration issue](screenshots/16-error-solved-nginx-syntax-ok.png)

### Phase 8 — Proving Load Balancing

I tested requests through the Nginx entry point and used the backend identifiers to observe which server returned each response.

The manual's local test is:

```bash
for i in {1..10}; do
    curl -sS http://localhost/lb-test.txt
    printf '\n'
done
```

This is a functional test of repeated request handling and backend selection. Ten requests are useful for a demonstration, but they are not a throughput or capacity benchmark.

**Reverse proxy and load-balancer evidence**

![Nginx reverse proxy and load-balancer setup](screenshots/17-nginx-reverse-proxy-and-as-lb.png)

**Traffic-distribution evidence**

![Responses Demonstrating Load Balancing](screenshots/18-proof-of=load-balancing.png)

**Repeated-request test**

![Ten-request load-balancing test](screenshots/19-testing-lb-with-10-request-successful.png)

### Phase 9 — Observing Backend Failure Behaviour

I included a test with AppNode1 unavailable to investigate how the setup handled a failed upstream.

The exercise described in the manual stops Apache on one backend, sends requests through Nginx, and then restores the service. It demonstrates why multiple backends can improve resilience to an application-server failure.

The result needs to be interpreted within the configuration and test conditions. Passive failure handling can involve unsuccessful attempts before a backend is temporarily treated as unavailable. A single failure exercise does not establish zero dropped requests or a production availability guarantee.

**Backend failure evidence**

![Load-balancer observation with AppNode1 unavailable](screenshots/20-lb-detects-failed-appnode1-failed-backend.png)

### Phase 10 — Adding an Elastic IP

I associated an Elastic IP with the Nginx instance to give the public endpoint a stable address for the domain configuration.

After the address change, I checked the service and browser response through the Elastic IP. This confirmed the public entry point before adding domain-based access.

**Elastic IP association**

![Elastic IP allocated to the Nginx instance](screenshots/21-ec2-elastic-ip-allocated-to-lb-instance.png)

**Nginx service check**

![Nginx status on the Elastic IP server](screenshots/22-ngix-status-on-elastic-ip=server.png)

**Public IP browser test**

![Browser access through the Elastic IP](screenshots/23-elastic-ip-showing-on-browser.png)

### Phase 11 — Connecting the Domain

I connected the domain to the load-balancer endpoint and tested domain-based access.

The root-domain DNS pattern is:

| Record | Name | Target |
|---|---|---|
| A | `@` | Load-balancer Elastic IP |

The manual also describes a `www` hostname using an A record or a CNAME. That hostname should be validated separately if it is included in the certificate request.

The Nginx `server_name` needs to match the hostname being served. Public DNS resolution, HTTP access, and the server configuration should be working before requesting the certificate.

Useful checks include:

```bash
dig +short YOUR_DOMAIN
curl -I http://YOUR_DOMAIN
curl http://YOUR_DOMAIN/lb-test.txt
```

**Elastic IP and domain response**

![Nginx response through Elastic IP and DNS](screenshots/24-nginx-lb-elastic-ip-dns.png)

**Browser access through the domain**

![Apache response through the domain in the browser](screenshots/25-dns-domain-apache-on-browser.png)

**HTTP domain test**

![Domain test over HTTP](screenshots/26-domain-test-over-http.png)

**Backend identifier through the domain**

![Backend identifier accessed through the domain](screenshots/27-domain-backend-identifier-file-test-successful.png)

**Additional protocol-response test**

![Additional domain HTTP and HTTPS response test](screenshots/28-domain-https-test-showing-http-success.png)

**Nginx service state during domain testing**

![Nginx running during domain testing](screenshots/29-domain-lb-nginx-status-running-successful.png)

**Domain HTTP response-chain test**

![HTTP response-chain test through the load balancer](screenshots/30-domain-lb-http-chain-successful.png)

### Phase 12 — Installing Certbot and Enabling HTTPS

After establishing domain-based HTTP access, I prepared the Snap environment and used Certbot's Nginx integration for the certificate workflow.

Certbot acts as the ACME client. Let's Encrypt acts as the Certificate Authority. These are separate responsibilities: Certbot requests and manages the certificate, while Let's Encrypt issues it after successful validation.

The installation and request pattern in the manual is:

```bash
sudo snap install --classic certbot

sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot

sudo certbot --nginx \
    -d YOUR_DOMAIN \
    -d www.YOUR_DOMAIN \
    --redirect
```

Only request hostnames that resolve correctly and are intended to be served by this deployment.

**Snap preparation**

![Snap socket setup](screenshots/31-snap-socket-installed.png)

**Certbot installation**

![Certbot installation through Snap](screenshots/32-certbot-nginx-snap-installed.png)

**Nginx certificate integration**

![Certbot Nginx integration request
](screenshots/33-cerbot-nginx-integration-request.png)

### Phase 13 — Testing Redirection and HTTPS Load Balancing

I checked that HTTP redirected to HTTPS and that the secured endpoint still served requests through the backend pool.

This matters because successful certificate installation alone does not confirm that the application request path still works.

Relevant checks are:

```bash
curl -I http://YOUR_DOMAIN
curl -I https://YOUR_DOMAIN
```

For backend selection:

```bash
for i in {1..10}; do
    curl -sS https://YOUR_DOMAIN/lb-test.txt
    printf '\n'
done
```

**HTTP-to-HTTPS redirection**

![HTTP-to-HTTPS redirection test](screenshots/34-certbot-http-https-redirection-test-successful.png)

**HTTPS and backend-distribution test**

![HTTPS and load-balancing test](screenshots/35-https-and-load-balancing-test-successful.png)

### Phase 14 — Testing Certificate Renewal

I tested the renewal workflow using a dry run:

```bash
sudo certbot renew --dry-run
```

The dry run checks whether the renewal process can complete without waiting for the production certificate to approach expiry.

A successful dry run is separate from confirming that the scheduled renewal mechanism is installed and enabled. The manual recommends inspecting the existing timer before adding another scheduler:

```bash
systemctl list-timers --all
```

**Renewal test evidence**

![Successful Certbot renewal dry-run test](screenshots/36-certbot-renew-dry-run-test-successful.png)

## Troubleshooting and Recovery

I kept the failure screenshots because they explain how I investigated the deployment. They show that the final result involved diagnosis and revalidation, rather than a sequence of commands that worked on the first attempt.

### 1. Public and Private Address Confusion in the Hosts-File Workflow

I documented an issue involving public IP addresses in the backend hosts-file setup.

The intended architecture uses private addresses for communication from Nginx to the Apache servers. My investigation therefore needed to check that each backend name resolved to the intended private address and that the load balancer could reach it.

Public IP use is not, by itself, proof of a particular Nginx syntax error. Address selection, hostname resolution, network reachability, and configuration syntax are separate checks.

The useful investigation sequence is:

```bash
cat /etc/hosts

getent hosts Web1
getent hosts Web2

curl -v http://Web1/lb-test.txt
curl -v http://Web2/lb-test.txt
```

**Address-mapping error evidence**

![Address-selection issue in the hosts-file workflow](screenshots/37-hosts-file-address-error.png)

The hosts-file and subsequent backend-response checks appear in screenshots 09, 12, and 14.

**What I learned:** The load balancer must resolve and reach the correct backend address. A healthy Apache service is only one part of that path.

### 2. Failed Nginx Configuration Validation

I encountered a failed Nginx configuration test and retained the failure evidence alongside the successful follow-up check.

The diagnostic starting point is:

```bash
sudo nginx -t
```

The output should guide the investigation. Depending on the reported issue, relevant checks can include directive placement, braces, semicolons, upstream names, included files, and duplicate listener configuration.

Additional diagnostic commands are:

```bash
sudo journalctl -u nginx -n 100 --no-pager
sudo tail -n 100 /var/log/nginx/error.log
```

These are troubleshooting references; this documentation does not assign an exact error message or root cause that is not readable in the evidence.

**Failed configuration check**

![Failed Nginx configuration validation
](screenshots/38-error-nginx-config-test-failed.png)

The successful follow-up validation Resolved.

![Successful Follow-up Validation](screenshots/16-error-solved-nginx-syntax-ok.png)

**What I learned:** I should validate the configuration before reloading the service, then test requests after the reload. Syntax validity and working request routing are different checkpoints.

### 3. Backend Unavailability

The backend failure exercise helped me distinguish proxy availability from upstream availability.

Nginx can be running while a backend service is stopped or unreachable. Investigating that condition requires checking the backend service, the network path, and the upstream configuration.

The failure test is documented in screenshot 20.

**What I learned:** A service-status check cannot replace an end-to-end request test. I need to understand where the request stops.

## Validation Matrix

| Layer | Validation purpose | Screenshot evidence |
|---|---|---|
| Infrastructure | Establish the three-instance environment | 01 |
| Backend access | Access both Ubuntu backend servers | 02, 04 |
| Backend services | Inspect Apache setup and running state | 03, 05, 11, 13 |
| Local application | Check individual backend responses | 06, 07, 08 |
| Load-balancer access | Access the Nginx server | `08-loadbalancer-ubuntu-login.png` |
| Name resolution | Review backend mappings | 09, 37 |
| Backend network path | Obtain backend responses from the load balancer | 12, 14 |
| Nginx configuration | Validate syntax and document recovery | 10, 16, 38 |
| Proxy service | Inspect Nginx running state | 15, 22, 29 |
| Load balancing | Observe backend selection and repeated requests | 17–19 |
| Backend failure | Investigate an unavailable upstream | 20 |
| Stable public endpoint | Associate and test the Elastic IP | 21–23 |
| Domain access | Check domain-based responses | 24–30 |
| Certificate setup | Prepare Snap and integrate Certbot | 31–33 |
| HTTPS | Check redirection and secured request handling | 34, 35 |
| Renewal | Test the certificate-renewal workflow | 36 |

Two files share the `08` prefix. Their complete filenames distinguish the backend test from the load-balancer login screenshot.

## Security Design and Operational Boundaries

### Intended Traffic Rules

The manual specifies the following security-group design:

| Destination | Port | Intended source |
|---|---:|---|
| Nginx instance | 22 | Administrator's public IP |
| Nginx instance | 80 | Public clients and HTTP certificate validation |
| Nginx instance | 443 | Public HTTPS clients |
| Apache backends | 80 | Nginx load-balancer security group |
| Apache backends | 22 | Approved administration path |

Referencing the Nginx security group in the backend HTTP rule limits direct application access more cleanly than permitting all internet sources.

Dedicated security-group screenshots are not included in the current evidence set. This table explains the intended rules rather than claiming an independently verified final firewall configuration.

### Availability Boundaries

The two Apache servers provide multiple application backends, but there is only one Nginx instance. The load balancer remains a single point of failure.

The backend failure exercise demonstrates behaviour under the tested condition. It does not prove resilience to load-balancer failure, an Availability Zone outage, or every possible application failure.

### Certificate and Repository Handling

SSH private keys, AWS credentials, access tokens, and TLS private keys do not belong in the public project documentation.

A sanitized Nginx example can reference certificate paths without including the certificate's private-key contents.

## Results

The project documentation follows the request path from individually tested Apache servers to a domain-based HTTPS entry point.

The evidence covers:

- Independent backend preparation and response checks.
- Nginx service and configuration validation.
- Reverse proxying and observable backend distribution.
- A repeated-request functional test.
- Backend failure observation.
- Elastic IP and domain access.
- HTTP-to-HTTPS redirection.
- HTTPS load-balancing validation.
- Certificate-renewal testing.
- Address-mapping and Nginx configuration troubleshooting.

The screenshots are implementation evidence, not measurements of production capacity. This project does not claim a throughput figure, latency target, uptime percentage, or availability SLA.

## What I Learned

The most useful lesson was to troubleshoot in layers.

First, I needed the backend application to respond locally. Next, I needed the load balancer to resolve and reach each backend. Only then did it make sense to test Nginx routing, the public address, domain access, and HTTPS.

This approach made the investigation more focused. A backend connectivity problem belongs to a different layer from a DNS problem. A failed Nginx configuration test needs a different response from an expired certificate or a stopped Apache process.

I also learned why identifiable responses are valuable. They turn load balancing from a configuration claim into something I can observe through requests.

Finally, I learned to treat certificate management as an ongoing responsibility. HTTPS setup is followed by renewal testing and scheduler verification, not just a successful browser visit.

## Skills Demonstrated

| Skill area | How this project demonstrates it |
|---|---|
| Linux administration | SSH access, service inspection, configuration editing, and command-line testing |
| AWS infrastructure | Working with EC2 instances and an Elastic IP |
| Networking | Separating public addressing, private backend communication, local name resolution, and public DNS |
| Web infrastructure | Connecting Apache backends through an Nginx reverse proxy |
| Load balancing | Using an upstream pool and identifiable responses to test distribution |
| TLS operations | Configuring HTTPS, checking redirection, and testing certificate renewal |
| Troubleshooting | Preserving failure evidence and revalidating after corrections |
| Technical documentation | Explaining the request path and linking evidence to implementation stages |

## Future Improvements

The next step is to make the deployment more reproducible and easier to operate.

Planned improvements include:

- Exporting sanitized configuration files into a `configs/` directory.
- Adding reusable backend and HTTPS test scripts.
- Provisioning infrastructure with Terraform.
- Managing server configuration with Ansible.
- Adding logs, metrics, and alerts.
- Recording dedicated security-group and DNS-resolution evidence.
- Verifying and documenting the certificate-renewal scheduler.
- Adding backend HTTPS where required.
- Testing load-balancer redundancy.
- Running controlled performance tests with measured results.
- Documenting recovery and resource-cleanup procedures.

These are future improvements, not features already demonstrated by the current repository.

## References

- [NGINX HTTP Load Balancing](https://nginx.org/en/docs/http/load_balancing.html)
- [NGINX Upstream Module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [AWS Elastic IP Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- [Certbot Documentation](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [Let's Encrypt: Keeping Port 80 Open](https://letsencrypt.org/docs/allow-port-80/)