---
title: Why Enterprise Authentication is Worth Setting Up in a HomeLab
author: Brian Grimmett
date: 2025-06-21 12:00:00 -0600
categories: [HomeLab, Authentication]
tags: [homelab]
render_with_liquid: false
---

## Intro
Playing around in a lab is one of the most beneficial things I have done in my time within the Enterprise IT industry. Being able to replicate customer environments for application deployment, test automation, or even debugging product issues; the homelab has become as important as a keyboard, mouse, or monitor. 

However, one of the issues with creating a homelab is the constant improvement and backlog that gets created for essentially managing another IT environment after working within one all day. One of the biggest challenges I have is authentication and *aspiring* to not have my passwords along the line of 'password123' or 'pleasework!'.

Notice how with Google accounts, you can have one-button login to many different services, this integration allows the end-user to have minimal friction to logging into day-to-day services. Replicating this in a homelab can take some setup, but the benefit of having a singular account that can access all machines, services, and applications with the password being managed through a singular service makes the management of this hobby much easier.

## What is Enterprise Authentication
So, what is Enterprise Authentication? At a basic level the [University of Texas Enterprise Technology](https://iamservices.utexas.edu/solutions/authentication/) defines authentication "is the act of determining that someone is who they claim to be". Where enterprise authentication can be the act of determining persons are who they claim to be at scale. Where enterprise authentication can help applications and services determine people are who they say they are for access and use of these systems.

Two terms to think about are local accounts versus social accounts, where a local account can be thought as scoped to a specific thing (application, linux box, etc) and social accounts can be mapped as users to the thing or many other things when the account is managed by an external identity provider. Local accounts can be thought as the user you create with `useradd NAME` on the linux command line and the user is added to the `/etc/passwd` file. When social accounts will authenticate via SSSD and by default no entry is created in `/etc/passwd`. SSSD is the System Security Services Daemon where it accesses remote directories and authentication methods, this is how the system can determine the user is who they say they are AND they have access to that system via reading the directory. 

When I first started in the homelab I was using local accounts everywhere, keeping track of all these usernames and passwords became a nightmare of a Notepad++ file, which felt wrong and unmanageable, especially as a I expanded my "production services" for friends and family to access. When I first started looking into what can I use in a lab, I cam across the terms LDAP and SAML; where LDAP is Lightweight Access Directory Protocol and SAML is Security Assertion Markup Language. Both LDAP and SAML are two types of authenticators but the difference is how the authentication is happening. 

How to determine the right tool for the job will be out of scope of this article, if you are interested in learning more in LDAP and SAML check out this great video by [JunmpCloud](https://www.youtube.com/watch?v=_NCcLJin30E)

## Why is it Worth Setting Up
After going through all the definitions and explanations, let's get to the real question: Why should you bother setting enterprise authentication in my homelab?

### Ease of Use
Once setup, having a single sign-on or social account that is one username/password to login is beyond useful. For example, I use a single-user with the password stored in my password manager and since all my services/applications are under the starbase.icu domain, then the username autofill for all my services. As part of my security management I have Ansible automatons built around rotating this password through my IAM and updating the password manager entry every 30 days. To do this with local accounts would either be much more complex automation or a real pain manual process for each of my users.

### Scalability
Personally, after spending the time and building the lab I have, I aspire to open my services to my friends and family. For example, let's take a look at NextCloud. [NextCloud](https://nextcloud.com/about/) is an open-source content platform which I use a self-hosted option for Dropbox or OneDrive. Personally, I use NextCloud as my home directory on all my systems, since I daily drive multiple operating systems and this makes it to where my data is everywhere I need it to be. With this tool being so useful and integral to my workflows now, I wanted to open the service to family and friends where my lab can be an on-prem, in control of their data than a cloud platform which is charging a recurring subscription.

Once the hardware and the infrastructure was highly available and able to be consumed by multiple users (the illusion of HA, this is a homelab!), the next question became how do I onboard users into the environment and ensure RBAC is enforced so no one deletes critical data or brings down "prod". Enterprise authentication became the option where I created an LDAP group `nextcloud-admins` and `nextcloud-users`, where once a user is onboarded to either group they can log into NextCloud with the access needed to consume the application. As well since the user is created any new applications that integrate with social authentication can have the uses(s) added to a new group `app_name-users` or `app_name-admins` so they can now consume the new application!

### Real World Practice
Doing this via LDAP and not local accounts through NextCloud allow for scalability as more services become available as apart of the homelab. This management as well is giving IAM practice to the lab, and being able to troubleshoot and build management around a 'true end user' environment. 

## What are the Options?
All this sounds great, right! Now what can you integrate with that is free, easy to setup, and applicable to real-world IT. Personally I use two identity providers, Okta and Red Hat IdM (Identity Management). [Okta](https://www.okta.com/) works great as a free SAML provider through their developer program found [here](https://developer.okta.com/). You can create an organization, groups, create SAML applications, as well as integrate with their MFA tool which many enterprises use such as [jetBlue, FedEX, Hewlett Packard, and many more](https://www.okta.com/customers/). Where [Red Hat IDM](https://access.redhat.com/products/identity-management), though not a free software, was most applicable to me as a Red Hat employee as an LDAP provider; however, [FreeIPA](https://www.freeipa.org/) is a viable alternative to achieve the same functionality, just without some of the extra services IdM provides. Personally, if you are just getting started with enterprise authentication Okta is a great place to start with their documentation and public tutorials of how to integrate with many applications (check the sources below)!

## Conclusion
On a personal note, thanks for reading this far and I hope you learned something and added yet another thing to the lab backlog. Enterprise authentication is not something just for big companies can can make your life easier and less management in a homelab! If you have any questions, comments, or concerns please feel free to leave a comment below!

## Sources
- [Univeristy of Texas Enterprise Technology - Authentication](https://iamservices.utexas.edu/solutions/authentication/)
- [NextCloud - About Us](https://nextcloud.com/about/)
- [Okta Developer](https://developer.okta.com/)
- [Red Hat Docs - What is SSSD?](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_authentication_and_authorization_in_rhel/understanding-sssd-and-its-benefits_configuring-authentication-and-authorization-in-rhel) 
- [JumpCloud - The Difference Between LDAP and SAML SSO](https://jumpcloud.com/blog/difference-ldap-saml-sso#:~:text=LDAP%20and%20SAML%20are%20both,between%20using%20LDAP%20or%20SAML)
- [FreeIPA](https://www.freeipa.org/)
- [CloudFlare - What is SAML?](https://www.cloudflare.com/learning/access-management/what-is-saml/#:~:text=Security%20Assertion%20Markup%20Language%2C%20or,that%20authentication%20to%20multiple%20applications)

### Tutorials
- [Okta - Add an Okta SAML application](https://help.okta.com/oag/en-us/content/topics/access-gateway/add-app-saml-pass-thru-add-okta.htm)
- [NextCloud - Setup SSO with Okta](https://portal.nextcloud.com/article/Authentication/Single-Sign-On-(SSO)/Nextcloud-Single-Sign-On-with-Okta)
