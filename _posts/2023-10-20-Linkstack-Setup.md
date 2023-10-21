---
title: Setting up my Linkstack
aside: true
categories:
- Raspberry Pi
- Home Server
feature_image: "/assets/pictures/DSC_0023.jpg"
---

Setting up my Linkstack:
<!-- this part ^^ is how much shows in the description of the post by using a parachgraph format it autmoatically pics how much to show -->
<!-- more -->

This past week I embarked on a journey to make and host my own Linkstack. [@Michael Lamb]( https://michaellamb.dev/) thank you for the idea! Michael had warned me I would need to use NGINX as my proxy manager on the Pi. I have not had good luck in setting up this tool in the past.  I am going to tell y’all what I had to do to get it up and running to open up my Linkstack to the web. I had an existing docker image for NGINX on my Pi. I tried restarting but kept getting an Internal Error when trying to start the container. I then removed the container and tried pulling the image down again and starting fresh using this [guide]( https://nginxproxymanager.com/guide/#quick-setup).  With this done I was able to get the container and app started. My next issue was enabling and forcing SSL on the proxy it would error every time I set that on my proxy for my Linkstack. I knew I had certbot already installed on my pi but when I went to check the certs I had previously had, and they were not there. I went to regenerate them, but I ran into port 80 errors after turning off the NGINX app. In order to resolve this I used these two commands.

```
sudo ss -tlpn | grep -E ":(80|443)"^C
sudo systemctl stop nginx
```

With this done I was able to use this command

```
sudo certbot certonly --standalone -d links.jackmazza.xyz
```
This allowed me to generate a cert for my domain. This still didn’t get me my working product. Next I had to go mess with cloud flare settings. Turn off Always Use HTTPS - setting found SSL/TLS, Edge Certificate. Then go to Rules and add the following below but replace the domain with your own.
```
Rule 1:
http://*yourdomain/.well-known/acme-challenge/*
cache level standard
Rule 2:
http://*yourdomain/*
Always use https
```
Then I turned off the proxy setting on the A record in the Cloudflare DNS settings. With all of these steps complete the [link](https://links.jackmazza.xyz/@jackmazza) finally worked.

Here are some of the useful links I used to set this up and figure it all out.
[Linkstack Guide]( https://www.youtube.com/watch?v=s1rv0bOGcZ4)
[Lets Encrypt Troubleshooting]( https://www.reddit.com/r/selfhosted/comments/oe4dl6/nginx_proxy_manager_getting_internal_error/)
[Certbot Setup]( https://www.youtube.com/watch?v=s1rv0bOGcZ4)
[Cloudflare Troubleshooting]( https://community.letsencrypt.org/t/renew-lets-encrypt-cert-issued-with-cert-bot-behind-cloudflare/57450/7)
[Certbot Setup]( https://www.youtube.com/watch?v=s1rv0bOGcZ4)


