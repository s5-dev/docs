# Using DNSLink with your S5 node

## Introduction

[DNSLink](https://dnslink.dev/) is a way to host static websites on a standard domain like blog.your.domain, while using a nonstandard backend (Like S5 or IPFS). In this tutorial, I will show you how to use the [S5 Deploy](https://github.com/s5-dev/s5_deploy) tool to deploy your own static website to S5.

## Guide

### Step 1: Decide what content you want to host

For this example I'm going to be using the absolute most basic thing possible, a plain html file. Keep in mind though that this kind of "static hosting" can be anything from as simple as a single html file, to as complex as a full canva-view flutter app. Static doesn't mean useless. But for this example, I created a folder that looks like this:

```
web:
=> index.html
```

Inside that document:

```html
<h1>hello!</h1>
```

### Step 2: Install S5 Deploy

S5 deploy is a CLI app designed specifically for doing this kind of static deploy on the S5 network (very simmilar to [IPFS deploy](https://github.com/ipfs-shipyard/ipfs-deploy)).

To install run this in your terminal:

> :warning: S5-Deploy is currently only built to work for x86-linux distros

```bash
bash <(curl -s https://raw.githubusercontent.com/s5-dev/s5_deploy/main/install.sh)
```

Now run a version check to make sure everything is working properly like so:

```bash
📦[covalent@debian s5-docs]$ s5_deploy --version
Version: 0.1.0
```

### Step 3: Deploy Your Site to S5

S5 Deploy is by zero config application, so while you can specify your own node, data keys, etc you don't have to.

So run `s5_deploy` and point it to the correct directory. Remember to accept the DNSLink instructions. It should look something like this:

```bash
📦[covalent@debian test-site]$ s5_deploy web/
✔ Setting things up...
✔ Initializing S5...
✔ Scanning web/
✔ Uploading to S5...
✔ Updating resolver link...
Sucsesss!
Static Link: s5://z31rYcmNFWQ8WiWKFB1eUrS5UZLt1LDfKm88t73LHMdmNMfX
Resolver Link: s5://zrjRkGHwnSLCdDRzrKgyTjPZu1vGHYEqW21HLWvR1cH3Ejj
Resolver Link Subdomain: https://bexw5q7kl6yh2ttbn5sgcobfmioceszuq5e54idqnan2azxngi4sh3mq.s5.ninja
NOTE: This subdomain may not work depending on wildcard permissions
Would you like DNSLink instructions? (Y/n): y
Please put this TXT record on your domain: dnslink=/s5/zrjRkGHwnSLCdDRzrKgyTjPZu1vGHYEqW21HLWvR1cH3Ejj
```

To make sure everything is working, paste the resolver link into a [CID explorer](../tools/cid-one.md).

> Make sure to remove the s5:// prefix or cid.one will get confused.

### Step 4: Configure Your S5 Node

While in theory you could view your page now, none of the public S5 nodes allow DNSLink in the allow scope so they don't get burdened hosting websites for free. The good thing is it's extremely easy to set up and configure your own download-only node:

1. Install S5 [install](../install/README.md).
2. Install and setup [caddy](../install/caddy.md). This should be a reverse proxy to your domain name, so when someone searches your static site it gets forwarded propelry so for example, if your website is dnslinktestsite.jptr.tech, your reverse proxy should look like this:

```
dnslinktestsite.jptr.tech {
    reverse_proxy localhost:5050
}
```

3. Add `s5/subdomain/load` to the [alwaysAllowedScopes](../install/config/README.md#accounts). Note, you do not need to enable full account access, just the subdomain load option.

### Step 5: Configure DNS

Configuring DNS is always a bit of a pain as _every_ provider does things differently. But in essence you need to do two things. First, point your domain to the IP of the S5 Node. Second, add a DNSLink TXT record so your node knows where to look for your site.

So let's say your blog is at dnslinktestsite.jptr.tech. You need two DNS entries:

| Type | Name                      | Content                                                     |
| ---- | ------------------------- | ----------------------------------------------------------- |
| A    | dnslinktestsite           | 23.493.4.55                                                 |
| TXT  | \_dnslink.dnslinktestsite | dnslink=/s5/zrjRkGHwnSLCdDRzrKgyTjPZu1vGHYEqW21HLWvR1cH3Ejj |

So once those registry entrys propagate, your site should work! View the test site from this guide [here](https://dnslinktestsite.jptr.tech/).
