# Linux Server Hardening Guide

After working for some time as an engineer at a very security-obsessed company I decided it would be a good exercise to note down and organise all the things I've learned both at work and in my spare time related to Linux server security. This guide will focus on Linux in a server context but many of the ideas here are applicable to other systems.

## Guiding principles

1. Defence in depth. No single control is going to save you, you want to build strong security into as many levels of your systems as possible so that the odds are if you do get attacked you'll have something that breaks the attack chain.

2. Threat model. Decide what level of attacks you care about defending against and which you're not going to worry about. If you're running a test nodejs server on a tiny vps and have no sensitive data on it at all, you probably don't need to try to defend against the potential for a nation-state attack.

3. Security is always a trade-off with convenience. Before securing a system decide how much inconvenience you're willing to tolerate in the name of security. Yubikeys are powerful tools but they're also hassle to use.

4. This is a living document and PRs are very welcome.

## Basics

These are things that you should *always* even if you don't care a lot about the security of the system.

1. Turn off unnecessary internet-facing services.
2. Use a basic ingress firewall like iptables.
3. Disable root ssh access.
4. Use ssh keys for authentication and disable password authentication.
5. Don't run ssh on tcp/22, use a random high-number port instead.

## Basic hardening

Quick and easy ways to harden a server:

1. Remove the +s (suid) bit from all system binaries that you don't need to use.
2. Restrict ssh ingress to specific networks or IPs.
3. Use a TPM or secure enclave to store your ssh key so that the private key can't be read by humans. On macs this can be done with Secretive - https://github.com/maxgoedjen/secretive. On iOS some SSH clients support generating ssh keys in the secure enclave. For a generic solution this can be done with a Yubikey or similar hardware security key.

## Two-factor SSH/sudo authentication

I highly recommend using the duo pam module - https://duo.com/docs/duounix - for sshd and sudo access. It's very convenient, provides a much greater level of security than ssh keys alone and is free for personal use.

1. Compile and install the pam module
2. Create a Unix application on the duo website
3. Configure /etc/duo/pam\_duo.conf with your duo application config, set autopush = yes
4. Add this line to the top of your /etc/pam.d/sshd file:

````
auth sufficient /usr/lib64/security/pam_duo.so
````

(make sure the path to the installed module is correct)

5. Configure sshd:

````
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
````

You can exclude specific users from needing to use duo with:

````
Match User myuser
  AuthenticationMethods publickey
````
