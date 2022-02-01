# Linux Server Hardening Guide

After working for some time as an engineer at a very security-obsessed company I decided it would be a good exercise to note down and organise all the things I've learned both at work and in my spare time related to Linux server security. This guide will focus on Linux in a server context but many of the ideas here are applicable to other systems.

## Guiding principles

1. Defence in depth. No single control is going to save you, you want to build strong security into as many levels of your systems as possible so that the odds are if you do get attacked you'll have something that breaks the attack chain.

2. Threat model. Decide what level of attacks you care about defending against and which you're not going to worry about. If you're running a test nodejs server on a tiny vps and have no sensitive data on it at all, you probably don't need to try to defend against the potential for a nation-state attack.

3. Security is always a trade-off with convenience. Before securing a system decide how much inconvenience you're willing to tolerate in the name of security. Yubikeys are powerful tools but they're also hassle to use.
