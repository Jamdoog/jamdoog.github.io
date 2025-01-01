---
title: Configuring FreeIPA LDAP with Ansible Automation Platform 2.5
slug: configuring-freeipa-ldap-with-ansible-automation-platform-25
date_published: 2024-12-31T08:11:45.000Z
date_updated: 2025-01-01T00:34:00.000Z
tags: Linux, Ansible, RedHat, FreeIPA
---

## Introduction

This is a quick blog post to highlight integrating FreeIPA's LDAP(S) into the recent Ansible Automation Platform 2.5.

This also applies to RedHat's Identity Management (IDM) which is their commercial version of FreeIPA.

While documentation of the LDAP provider exists, I could only find 1 example of this online from a few years ago.

This post is more of a reference point then a tutorial.

## Configuring

- Create a bind user and give it appropriate permissions to search the database
- (Optional) Create a user group that is required for someone to log into the platform with
- (Optional) Create a user group that will grant administrative privileges

Once on the platform, you can use this configuration as a point of reference:

![LDAP Configuration Image](/assets/images/freeipa-ansible/ldap-config-1.png)
![LDAP Configuration Image](/assets/images/freeipa-ansible/ldap-config-2.png)

>**LDAPS may not work out the box if you do not trust the root certificate of the IPA server**

*Enroll your servers in your IPA realm. It will automatically fetch and trust the CA!*

## Group Rules

Rules in 2.5 has changed since 2.4, with it being a little less intuitive in my opinion. Here are the rules I deployed:

![LDAP Rules Image](/assets/images/freeipa-ansible/rules.png)

What this reads as:
- Name of the rule
- Trigger (In this case: A directory group)
- Operation (AND/OR) *This is ignored if there is only 1 provided*
- Groups (CN path to appropriate group)
- Revoke (Do the opposite. In this case; don't allow login)

Here is an example of a CN I used for the allow login group.

>cn=aap_users,cn=groups,cn=accounts,dc=jamdoog,dc=gra






