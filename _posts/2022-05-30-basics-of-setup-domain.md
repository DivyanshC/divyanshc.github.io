---
title: Setting up your new domain
date: 2022-05-30 18:33:00 +0530
categories: [homelab, github-pages]
tags: [homelab, servers] # tag names should always be lowercase
---

# Basics and setting up new domain

## What is a domain?

For Example for *https://www.divyashchauhan.com*

- **http/https** : protocols
- **www** : subdomain
- **divyanshchauhan** : domain
- **com** : domain extension

You can have one domain for your main website which is called apex domain like _`https://divyashchauhan.com`_ and then you can have subdomains for your other websites like _`https://blog.divyashchauhan.com/`_

So buy a domain name and then you can have subdomains for your other websites by pointing all these apex and subdomains to different servers ip addresses.

## Setting up cloudflare for your domain

Buy a domain from anywhere then create a cloudflare account then click on add site and then add the domain name in cloudlare dashboard.

It will give you cloudflare nameservers for your domain, then you have to add these to your custom DNS nameservers field in settings menu of the domain provider from which you purchased.

It will take some time for cloudflare. Ideally it should take around 10 minutes.

Cloudflare will show **Cloudflare is now protecting your site** in the dashboard for your domain.

Then go into DNS and you can see the **nameservers** we added will show up unser nameservers section.

Now in **DNS** section, if you have some a records pointing to some ip address then delete them and then create new a records and point them to your **servers ip addresses**.

## SSL/TLS Section

Select **Full (Srtict)** or Full Mode

If full strict is selected then check your website if invalid SSL error occurs then got with **full** mode.

## Email Forwarding

Can set up email forwarding for your custom domain email address like `help.divyanshchauhan.com` to your email address.
