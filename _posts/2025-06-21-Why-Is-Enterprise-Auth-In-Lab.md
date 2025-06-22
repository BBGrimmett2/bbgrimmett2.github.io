---
title: Why is Enterprise Authentication is Worth Setting Up in a HomeLab?
author: Brian Grimmett
date: 2025-06-21 12:00:00 -0600
categories: [HomeLab, Authentication]
tags: [homelab]
render_with_liquid: false
---

## Intro

Building and experimenting in a homelab has been one of the most valuable learning tools in my enterprise IT career. Whether I am replicating customer environments, testing automation, or even debugging product issues, the homelab has become as essential as my keyboard or monitor.

However, one of the issues with creating a homelab are the endless iterations and improvements. For me, one of the biggest challenges I had was authentication and *aspiring* to not have my passwords along the lines of 'password123' or 'pleasework!'.

Think about how Google accounts enable one-click access to countless services. This integration allows the end-user to reduce friction to logging into day-to-day services. Replicating that experience in a homelab takes effort, but having one account that gives access to all machines, services, and applications with centralized password management makes the whole environment easier to run.

## What is Enterprise Authentication

At a basic level, the [University of Texas Enterprise Technology](https://iamservices.utexas.edu/solutions/authentication/) defines authentication as "the act of determining that someone is who they claim to be." Think of enterprise authentication as doing this at scale... across dozens or hundreds of systems. It ensures users are correctly identified and authorized to access the tools, services, or environments they need, while simplifying that access for administrators and end users alike.

A local account is tied to a specific 'thing' (application, Linux box, etc); in contrast, social accounts can be mapped as users to the 'thing' or many other 'things' when the account is managed by an external identity provider. Enterprise authentication works by authenticating users to 'things' through an external identity provider. You can create local accounts with the `useradd NAME` Linux command, which adds the user to the `/etc/passwd` file Social accounts, on the other hand, are authenticated via SSSD and typically do not create entries in `/etc/passwd`. SSSD is the System Security Services Daemon, which accesses remote directories and authentication methods. This allows the system to determine that the user's identity and permissions are valid by querying the directory.

When I first started in the homelab, I was using local accounts everywhere, keeping track of all these usernames and passwords became a nightmare in a Notepad++ file; it felt wrong and unmanageable, especially as I expanded my "production services" for friends and family to access. When I first started looking into what I could use in a lab, I came across the terms LDAP and SAML, where LDAP is Lightweight Access Directory Protocol and SAML is Security Assertion Markup Language. Both LDAP and SAML are types of authenticators, but the difference is how the authentication happens.

How to determine the right tool for the job will be out of the scope of this article. If you are interested in learning more about LDAP and SAML, check out this great video by [JumpCloud](https://www.youtube.com/watch?v=_NCcLJin30E)

## Why is it Worth Setting Up

After going through all the definitions and explanations, let's get to the real question: Why should you bother setting up enterprise authentication in your homelab?

### Ease of Use

Once set up, having a single sign-on or social account that is one username and password to log in is beyond useful. For example, I use a single-user with the password stored in my password manager. Since all my services/applications are under the `starbase.icu` domain, autofill works seamlessly across them. As part of my security management, I have Ansible automations built around rotating this password through my IAM and updating the password manager entry every 30 days. To do this with local accounts would require far more complex automation or a tedious manual process for each of my users.

### Scalability

After building out my lab, I wanted to open up services to friends and family. For example, let's take a look at NextCloud. [NextCloud](https://nextcloud.com/about/) is an open-source content platform that I use as a self-hosted option for Dropbox or OneDrive. I use NextCloud as my home directory across all systems. Since I daily drive multiple operating systems, this setup ensures my data is where I need it and accessible. With this tool being so useful and integral to my workflows now, I wanted to open the service to family and friends where my lab can be an on-premise option, allowing for more control of their data than a cloud platform.

Once the hardware and the infrastructure were stable and (mostly) highly available, the next question became how do I onboard users into the environment and ensure RBAC is enforced so no one deletes critical data or brings down 'prod'. Enterprise authentication became the option where I created an LDAP group `nextcloud-admins` and `nextcloud-users`. Once a user is onboarded to either group, they can log into NextCloud with the access needed to consume the application. As well, since the user is created, any new applications that integrate with social authentication can simply be added to a new group like `app_name-users` or `app_name-admins` to gain access to that application too.

### Real World Practice

Using LDAP instead of local accounts through NextCloud makes it easier to scale users and services to the homelab. It also gives me hands-on experience managing IAM, troubleshooting user issues, and building processes around a true end-user experience.

For example, I was playing around with creating VLANs and firewalls across the different networks when I got messages from friends and alerts from my monitoring system that users were unable to log in to NextCloud. The issue turned out to be related to the network segmentation work, and the application was unable to reach the identity provider. Getting there took longer than I would like to admit, but that is the purpose of the lab. These steps and troubleshooting are comparable to what I have used professionally to deploy and validate LDAP and SAML configurations at scale.

## What are the Options?

All this sounds great, right? Exactly! To highlight some options, here are some that are free, easy to set up, and applicable to real-world IT. Personally, I use two identity providers, Okta and Red Hat IdM (Identity Management). [Okta](https://www.okta.com/) works great as a free SAML provider through their developer program (found [here](https://developer.okta.com/)). You can create an organization, groups, and SAML applications; Okta is the provider for many organizations enabling you to mimic real-world IT environments such as [JetBlue, FedEx, Hewlett Packard, and many more](https://www.okta.com/customers/). [Red Hat IDM](https://access.redhat.com/products/identity-management) is not free software, but it made sense for me to use as a Red Hat employee as an LDAP provider; however, [FreeIPA](https://www.freeipa.org/) is a viable alternative to achieve the same functionality but does lack some of the extra services IdM provides. If you are just getting started, I highly recommend beginning with Okta. Their documentation and public tutorials make integration with applications straightforward (references the sources).

## Conclusion

On a personal note, thanks for reading this far, and I hope you learned something and added yet another to-do to the lab backlog. Enterprise authentication is not just for big companies; it can make your life easier and reduce the management overhead in a homelab! If you have any questions, comments, or concerns, please feel free to leave a comment below!

## Sources

- [University of Texas Enterprise Technology - Authentication](https://iamservices.utexas.edu/solutions/authentication/)
- [NextCloud - About Us](https://nextcloud.com/about/)
- [Okta Developer](https://developer.okta.com/)
- [Red Hat Docs - What is SSSD?](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_authentication_and_authorization_in_rhel/understanding-sssd-and-its-benefits_configuring-authentication-and-authorization-in-rhel)
- [JumpCloud - The Difference Between LDAP and SAML SSO](https://jumpcloud.com/blog/difference-ldap-saml-sso#:~:text=LDAP%20and%20SAML%20are%20both,between%20using%20LDAP%20or%20SAML)
- [FreeIPA](https://www.freeipa.org/)
- [CloudFlare - What is SAML?](https://www.cloudflare.com/learning/access-management/what-is-saml/#:~:text=Security%20Assertion%20Markup%20Language%2C%20or,that%20authentication%20to%20multiple%20applications)

### Tutorials

- [Okta - Add an Okta SAML application](https://help.okta.com/oag/en-us/content/topics/access-gateway/add-app-saml-pass-thru-add-okta.htm)
- [NextCloud - Setup SSO with Okta](https://portal.nextcloud.com/article/Authentication/Single-Sign-On-(SSO)/Nextcloud-Single-Sign-On-with-Okta)
