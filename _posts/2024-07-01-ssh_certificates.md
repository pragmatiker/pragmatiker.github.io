---
layout: post
title: "SSH Certificates - Proof of Concept lab"
date: 2024-07-13 12:00:00 -0000
author: "TimDude"
categories:
  - IT Sec
  - ssh certificates
tags:
  - IT Sec
  - SSH
  - keycloak
  - OpenLDAP
  - Hashicorp Vault
---

SSH Certificates simplify secure access by using digitally signed certificates instead of traditional SSH keys. For admins, this approach offers centralized key management, making it easier to issue, expire, and revoke certificates. For users, it means more convenient authentication. By integrating FIDO2 authentication, users can securely authenticate to the SSH CA using hardware keys, eliminating the need for long passwords and enhancing security with a smooth, password-free experience.
  
Here i will build an SSH CA on Hashicorp Vault. The goal is for Clients to authenticate to Vault via FIDO2/Authn to obtain a signed SSH Certificate.
The moving parts will be containers with openLDAP, Keycloak and Hashicorp Vault.

---
**NOTE:**
The Lab is live on the internet. If you break it, at least leave me an issue in Github and tell me how you did it :)

---

{% assign sortedPosts = site.posts | sort: 'date' %}
{% for post in sortedPosts %}
  {% if post.categories contains page.categories[1] %} 
    {% unless post.title == page.title %}
    {% capture - %}{% increment my_counter %}{% endcapture %}
<li><a href="{{post.url}}" style="text-decoration:none;">Step {{my_counter}} - {{post.title}}</a></li>
    {% endunless %}
  {% endif %}
{% endfor %}
