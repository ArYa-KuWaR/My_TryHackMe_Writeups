Room Name : Takeover

Diffifculty : Easy

Room Link : https://tryhackme.com/room/takeover

## Introduction

This room introduces a common web vulnerability known as Subdomain Takeover. This occurs when a subdomain's DNS record points to a third-party service (like AWS or GitHub Pages), but the service is no longer configured or has been deleted. An attacker can then claim this orphaned service and "take over" the subdomain to serve malicious content.

## Step 1: Initial Reconnaissance (Nmap)

As with any target, the first step is to understand the attack surface. A simple nmap scan reveals the services running on the machine.

<img width="723" height="218" alt="Screenshot 2025-09-12 023520" src="https://github.com/user-attachments/assets/454be73e-d76c-46a9-ad82-7a9bcc74f551" />

The scan shows three open ports. Ports 80 (HTTP) and 443 (HTTPS) indicate a web server is active, which will be our primary focus. Port 22 (SSH) is for remote administration and is not relevant for this challenge.

## Step 2: Enumeration - Finding the First Subdomain

The room's description provides a hint that the company is rebuilding its support services. This strongly suggests the existence of a subdomain like support.futurevera.thm.
To access this, we must first tell our machine how to resolve this hostname by adding it to our /etc/hosts file. This file acts as a local DNS override

Navigating to https://support.futurevera.thm in a web browser presents a security warning for an invalid SSL certificate. This is a crucial clue. By inspecting the certificate's details, we can often find additional information.

<img width="1294" height="676" alt="Screenshot 2025-09-12 024304" src="https://github.com/user-attachments/assets/4c460692-9730-4705-b0bc-13fae303baff" />

## Step 3: Enumeration - Finding the Vulnerable Subdomain

Within the certificate's details, there is a field called Subject Alternative Name (SAN). This field lists all other hostnames for which the certificate is valid. Here, we find a second, more interesting subdomain.

<img width="1297" height="673" alt="hidden domain" src="https://github.com/user-attachments/assets/56f7c68e-ad36-4e1a-b466-854fc9150b0d" />

It looks like secret************.support.futurevera.thm.

We update our /etc/hosts file one more time to include this new discovery:

<img width="1049" height="547" alt="hidden sudo etc_host" src="https://github.com/user-attachments/assets/f71db423-8180-40e9-8dd9-3d94e4ec5c55" />

## Step 4: Exploitation - The Takeover

The final step is to navigate to this new secret subdomain. We find the flag when we use http instead of https as port 80 was seen open in nmap scan.

Navigating to http://secret************.support.futurevera.thm redirects us to an Amazon S3 bucket page with a "Not Found" error.

The flag is clearly visible in the new URL in the browser's address bar. The DNS for the secret subdomain correctly points to an S3 bucket, but that bucket has either been deleted or was never claimed. An attacker could now create an S3 bucket with that exact name to take control of the subdomain.

<img width="915" height="602" alt="Flag" src="https://github.com/user-attachments/assets/5444c968-0107-43f6-a5ec-d206f934ba26" />

## Conclusion 

This is the essence of a subdomain takeover. The company had a DNS record pointing to an Amazon S3 bucket, but they had either deleted the bucket or never configured it properly. This left the domain pointing to an empty, unclaimed spot on the internet that anyone (including an attacker) could claim to host their own content.

Hope this walkthrough was helpful! This was a really fun room, and it’s a great feeling when all the little clues click into place. Happy hacking!


