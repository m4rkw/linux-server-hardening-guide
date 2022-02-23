# Linux Server Hardening Guide

After working for some time as an engineer at a very security-obsessed company I decided it would be a good exercise to note down and organise all the things I've learned both at work and in my spare time related to Linux server security. This guide will focus on Linux in a server context but many of the ideas here are applicable to other systems.

## Contents

- [Guiding principles](#guiding-principles)
- [Basics](#basics)
- [Basic hardening](#basic-hardening)
- [Two-factor SSH and sudo authentication](#two-factor-ssh-and-sudo-authentication)
- [Restrict egress](#restrict-egress)
- [Use apparmor or selinux](#use-apparmor-or-selinux)
- [Harden webservers](#harden-webservers)
- [Monitor security config](#monitor-security-config)
- [Monitor for errant processes, listening ports and suid binaries](#monitor-for-errant-processes-listening-ports-and-suid-binaries)
- [Use canary tokens](#use-canary-tokens)
- [Keep an eye out for world-ending bugs](#keep-an-eye-out-for-world-ending-bugs)
- [Use immutable backups](#use-immutable-backups)
- [Be Devops-as-fuck](#be-devops-as-fuck)

## Guiding principles

1. Defence in depth. No single control is going to save you, you want to build strong security into as many levels of your systems as possible so that the odds are if you do get attacked you'll have something that breaks the attack chain.

2. Threat model. Decide what level of attacks you care about defending against and which you're not going to worry about. If you're running a test nodejs server on a tiny vps and have no sensitive data on it at all, you probably don't need to waste time hardening it against nation state hackers.

3. Security is always a trade-off with convenience. Before securing a system decide how much inconvenience you're willing to tolerate in the name of security. Yubikeys are powerful tools but they're also hassle to use.

4. Separate concerns. Put stuff you care a lot about far away from internet-facing stuff that you care less about. Don't run a personal mailserver on a machine that you regularly irc from.

5. This is a living document and PRs are very welcome.

## Basics

These are things that you should *always* even if you don't care a lot about the security of the system.

1. Turn off unnecessary internet-facing services.
2. Use a basic ingress firewall like iptables.
3. Disable root ssh access.
4. Use ssh keys for authentication and disable password authentication.
5. Don't run ssh on tcp/22, use a random high-number port instead.
6. Don't use NOPASSWD for sudo access.
7. Keep the system up to date.

## Basic hardening

Quick and easy ways to harden a server:

1. Remove the +s (suid) bit from all system binaries that you don't need to use.
2. Restrict ssh ingress to specific networks or IPs.
3. Use a TPM or secure enclave to store your ssh key so that the private key can't be read by humans. On macs this can be done with [Secretive](https://github.com/maxgoedjen/secretive). On iOS some SSH clients support generating ssh keys in the secure enclave. For a generic solution this can be done with a Yubikey or similar hardware security key.
4. If the machine isn't running any production services consider having a cron job that installs the latest distro packages for you. Most distros support a way to configure this.
5. If the machine has sensitive data on it, consider using disk encryption. Full-disk encryption is one option but a simpler way is to use cryptsetup to create an encrypted container.
6. Configure [fail2ban](https://www.fail2ban.org/wiki/index.php/Main_Page) for any public-facing services, especially any that use passwords for authentication

## Two-factor SSH and sudo authentication

I highly recommend using the [duo pam module](https://duo.com/docs/duounix) - for sshd and sudo access. It's very convenient, provides a much greater level of security than ssh keys alone and is free for personal use.

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
UsePAM yes
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
````

You can exclude specific users from needing to use duo with:

````
Match User myuser
  AuthenticationMethods publickey
````

You can also use this for sudo authentication by adding the "auth sufficient" line to the top of /etc/pam.d/sudo.

## Restrict egress

Everyone uses a basic firewall to restrict ingress but restricting egress is a very powerful control. Quite a lot of malware or exploits that try to get a foothold in a system will make calls out to the internet if they manage to gain code execution. Locking down your egress is a great way to block C2 channels and break the attack chain. It's not hard to do either:

1. Install squid proxy
2. Configure it to block everything not on an allowlist:

````
acl allowlist dstdom_regex '/etc/squid/allowlist.txt'
http_access allow allowlist
http_access deny all
````

3. Configure your firewall to block all egress except for DNS and outbound http/https as the user that the squid proxy runs as.
4. Configure all of your services that need egress to use the squid proxy (such as apt/yum etc).
5. Add regex rules as necessary to the allowlist.

## Use apparmor or selinux

Apparmor and selinux are an excellent way to have a last line of defence against 0day attacks. These tools allow you to create profiles that describe what each process running on your system is allowed to do, in terms of file access, system calls etc. Any attempt to read files, execute processes or do anything that isn't in the profile will be denied and a syslog message generated.

I prefer apparmor because it's a lot easier to use than selinux and comes with aa-logprof which parses /var/log/syslog for DENIED messages and then prompts you interactively to confirm or deny whether things should be allowed. If you allow them it automatically updates the apparmor profile for you which is really convenient. You can easily generate an apparmor profile for any system binary by simply creating an empty one and then iteratively running the program and aa-logprof to incrementally add the things it needs access to to its apparmor profile.

Ideally every process on your server that has any attack surface should have a carefully crafted apparmor profile. This means everything internet-facing and anything which could potentially possibly indirectly end up processing input from outside.

It's also a good idea to monitor syslog yourself and alert if you get DENIED messages for any reason. You may not find all of the relevant code paths in your initial testing so it's likely that at some later date a program with an actively-enforced apparmor profile might legitimately need to do something new which would then be blocked.

Carefully written apparmor/selinux profiles can be very effective at mitigating 0day exploits.

## Harden webservers

If you're running a webserver:

1. Use static content rather than scripts as much as possible. If you must use dynamic scripts, ideally just use them server-side to generate static content and then serve that rather than having dynamic scripts internet-facing.
2. Disable all modules that you aren't using.
3. Use mod security for apache or nginx and keep the rules up to date.
4. Use authentication for anything internet-facing that you don't want others to access.
5. Use TLS for everything and scan your TLS configuration with public scanners to verify that your configuration is strong.
6. If you're only running web services for your own use, consider running them on random high port numbers.

## Monitor security config

It's really easy to temporarily change a security control on one of your systems one day while debugging something and then forget to revert it. Monitoring of controls is really important. Ideally you want to make a list of all of the important controls that are enforced on your system, such as:

 - No passwordless sudo
 - sshd config correct
 - 2fa enforced for ssh
 - etc etc

Then have a monitoring system which checks these things are still in place every 5min or so and alerts you if they change. It's ok if you have a need to temporarily lift them for some reason as long as you get notified and remember to put them back.

## Monitor for errant processes, listening ports and suid binaries

If you get compromised by some kind of automated hacking bot, chances are it's going to try to persist or set up listening sockets. Periodically scan the process table for processes running under the uids of your public-facing services, such as the users that nginx or apache run as. Create a list of the processes you expect to see and alert for anything that isn't on the list. Note that processes can overwrite their own argv[0] to hide what they are so don't trust this, always check the /proc/{pid}/exe symlink.

Same thing for listening sockets, make a list of the listening ports you expect on each of your interfaces and alert for anything that appears that wasn't on the list. Do the same thing for suid binaries so you get alerted if package updates occur that revert any hardening work you've done.

Consider running [OSSEC](https://github.com/ossec/ossec-hids) which does a lot of this kind of security monitoring (and lots more) out of the box. And is surprisingly un-noisy.

## Use canary tokens

Thinkst Canary are an awesome company who provide free canary tokens here: https://canarytokens.org/generate

You can drop a few of these around your system and get alerted if anyone tries to use them. Good things to do this with are AWS keys and SSH keys. An attacker seeing these on a compromised system almost can't help trying them out and then you know straight away that you've got a problem. Thinkst's free tokens don't have an SSH key option but it's not hard to roll your own.

## Keep an eye out for world-ending bugs

These are becoming all too common these days, good places to keep an eye on are:

- Security mailing lists
- Security podcasts like the excellent [Risky Business](https://risky.biz)
- Popular infosec people on twitter

## Use immutable backups

Backup everything you care about on your servers to a remote object store such as S3 or B2. Both of these support object-locking, allowing you to specify a timeframe within which your backup objects cannot be modified or deleted. If someone compromises your server this gives you some protection against your machine being erased along with the backups.

Also don't forget to encrypt your backups, don't just trust encrypted buckets. [Rclone](https://rclone.org) - is a great tool for backups and supports many cloud backends. Also use zstd for compression as it's way faster than anything else. A simple way to create an encrypted backup would be something like:

````
tar -cP --exclude=/dev/* --exclude=lost\+found/* --exclude=/media/* --exclude=/proc/* \
  --exclude=/run/* --exclude=/sys/* --exclude=/tmp/* / /usr/bin/zstd | \
  /usr/bin/openssl aes-256-cbc -salt -pass file:/root/my-backup-key -md sha256 \
  -iter 5000 | /usr/bin/rclone rcat b2:bucket/myserver/2022-02-01.tar.zs.enc
````

## Be Devops-as-fuck

Don't build your machines manually, use provisioning tools like ansible. It can be challenging to keep the code in sync with reality but it's worth it. If you ever need to rebuild a system for any reason it's a lot easier when you have a bunch of ansible. I tend to use one of two approaches with my systems:

1. For machines that don't have a lot of changing data on them I build them purely with ansible and consider them ephemeral.

2. For machines that do have changing data I take tarball backups once a day and then have ansible code that builds the server which integrates with the backup system. The ansible playbook does some basic provisioning and then restores the latest backup from the backup service, copies a bunch of data from the backup into place, and then continues on with provisioning. This approach reduces some of the burden of keeping the ansible code in sync with the machine because the backups happen automatically and so your restore process always has access to the latest files and data from the machine.

If you feel the effort is justified, rebuilding your machines once a month will make attacker persistence practically impossible.
