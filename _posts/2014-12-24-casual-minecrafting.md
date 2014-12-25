---
layout: post
title: Casual Minecrafting
date: 2014-12-24 13:05
categories:
  - blog
---

When it comes to gaming, I'm what you would call "[cutting edge](http://xkcd.com/606/)". So naturally my kids and I have only recently started playing [Minecraft](https://minecraft.net/).

I saw [qrush](https://twitter.com/qrush) post something on twitter about running a custom server for weekends-only survival mode play, but what was really cool was the google maps-style map they had for it. Check it out at [pickaxe.club](http://pickaxe.club/).

My kids are young enough I don't yet want them playing on a public server, so the idea of a weekends-only-ish survival mode server for friends and family was appealing, and the map interface was just too cool to pass up. I know you can easily create a Minecraft Realms server, which has better server management integration and a hard to beat price point of $10/month, but man, _that map_. And, let's be honest here, the fun is usually in the process and not the result (for me).

So I bring you:

## How to set up your own vanilla Minecraft server with some cool add-ons in a bunch of moderately-easy steps

I tried to be reasonably thorough in the steps to follow, not assuming too much about any previous experience you might have, so skim and skip as you see fit.

I also originally set up my server using [Digital Ocean](https://www.digitalocean.com/) but I'm going to walk you through it using [Amazon Web Services](http://aws.amazon.com/) because it's cheaper, and frankly it's also just easier to handle a "sometimes-on" server in AWS.

If you don't care about the hosting of a server but just want to know how to set up and configure the maps, [skip ahead](#maps).

### Prerequisites

I assume you've got an AWS account and are at least somewhat familiar with how AWS works. It's pretty easy, so it doesn't take a ton of knowledge, but I will skip some of the generic AWS-isms to save time.

I also assume you've got a domain name of some sort for which you can control the DNS.

Let's get started with some basic security setup.

### Security

In AWS this is done through IAM roles (access) and EC2 Security Groups (networking).

Go to the EC2 area of the AWS console (no command line stuff here, sorry!) and under the "Network & Security" section choose "Security Groups". Click the "Create Security Group" button and set it up:

```
Name: minecraft
Description: Minecraft Server
VPC: [select your VPC in the list -- you likely only have one]
```

This Security Group is basically configuring a firewall. We'll need to allow SSH traffic, the minecraft server port, and I like to enable HTTP access as well for testing the map generation. (More on that later.) So add the following rules:

```
Type: SSH
Protocol: TCP
Port Range: 22
Source: Anywhere (let's live dangerously!)

Type: HTTP
Protocol: TCP
Port Range: 80
Source: Anywhere

Type: Custom TCP Rule
Protocol: TCP
Port Range: 25565
Source: Anywhere
```

Go ahead and create the group.

### IP Addresses

We'll want a consistent DNS name to access this server, and if you are hosting your DNS using Amazon's Route 53 service, you have more options available to you. Most of us probably aren't, though, so we need an Elastic IP so that it stays constant.

Quick aside: in AWS, when you stop and start an instance, it gets a new IP address assigned to it each time. So if we stop our Minecraft server on Monday morning, and start it back up on Friday evening, it'll have a new IP address unless we leverage an Elastic IP.

Under "Network & Security" in the EC2 console, choose "Elastic IPs". If you don't have any available, click the "Allocate New Address" button. I believe you get 5 with no questions asked. If you need a new one, just specify that it'll be used in `VPC` (and not `EC2`).

We can't use it until we have our server instance up and running, so let's do that now.

### Server Instance

Launch a new Ubuntu EC2 instance. I use the "Quick Start" AMI because, well, because I don't need anything special so "starting quickly" sounds good. The general recommendations online for running a Minecraft server are to have at least 2GB of memory, so we'll start there. For my server, I'll be hosting about 10 people max, so it's pretty light-weight. I don't want to spend a lot on this, so I'm using a `t2.small` instance size which gives me one CPU and 2GB of memory. This currently runs at $0.026 per hour. A `t2.medium` doubles your specs, and the price.

Make sure you specify the `minecraft` security group when creating it, and make sure to leave "Shutdown behavior" set to the default of `stop`. You want to keep this thing around.

After your instance has been launched, head back to "Elastic IPs" and associate your EIP with the newly launched instance.

### DNS

Set up an A record to point to your new instance. This is how people will connect to your server when they want to play, and setting it up now will make it easier to SSH in as you move forward also.

For example, you could set up something like `minecraft.example.com`.

### Quick Review

So we haven't even touched Minecraft yet, but now we have a server, with a dedicated IP address, and a tad bit more locked down network layer for access to the box. It's time to hop on and set up Minecraft.

### Java

Minecraft requires Java. Java doesn't ship on any OS by default, because how else would they ensure that you get asked "Do you not not not not not want to install the Ask Jeeves Yahoo Plus Plus Toolbar Extension, Spyware Edition?" [DEFAULT ANSWER IS YES, MEANING NO].

OpenJDK is really easy to install on Ubuntu, since the packages are available in the preconfigured repos. Some guides online say that you shouldn't use it to run a server due to "issues" but for right now, we're going to use it for ease. If you run into issues with it, [tweet at me](https://twitter.com/aaronlerch) and I'll update this post. [(Here are some additional helpful hints for installing Java on Ubuntu.)](http://minecraft.gamepedia.com/Tutorials/Setting_up_a_server#Ubuntu)

Install the OpenJDK JRE:

```
sudo apt-get update
sudo apt-get install openjdk-7-jre-headless
java -version
```

You should see something like `java version "1.7.0_65"` as a result. Good enough.

### MSM or Vanilla

You can just download and run a Minecraft server pretty easily. However we want the server to automatically start when the server boots, and we'd like it to stop cleanly when the server is going down, etc. The less I have to SSH into the server for, the better.

None of this is rocket science, but I found the [Minecraft Server Manager](http://msmhq.com/) to be pretty convenient at automating this, and some basic level of security (separate user account, etc.) Let's be clear: MSM does a *ton* of stuff, almost all of which we won't need. Just look at the [bullet list on their homepage](http://msmhq.com/) if you want to get a feel for it. The parts I want are the automatic setup and organization, so for this guide we'll go with MSM.

The [installation docs](http://msmhq.com/docs/installation.html) are pretty good, especially if you're the kind to trust the "wget this URL and execute it" setup helpers.

> Don't curl or wget scripts from the web to bash.
> \- me

Normally I think it's a terrible idea to just execute random stuff you've downloaded. However, in this case, I'm strongly favoring convenience over security given the nature of what I'm setting up, so let's just go for it. Accept the defaults except for the last question (which has a trick default answer if you aren't paying attention):

```
wget -q http://git.io/Sxpr9g -O /tmp/msm && bash /tmp/msm
```

```
MSM INSTALL: Configure installation
Install directory [/opt/msm]:
New server user to be created [minecraft]:
Add new user as system account? [y/N]: n
Complete installation with these values? [y/N]: y
```

Now we've got MSM installed and ready to be set up. There's just one teeny problem, MSM stopped pointing to the latest version of Minecraft a while back. But you can get around it pretty easily. The short version of the commands below is that MSM is also a Minecraft version manager, which lets you manage and run different versions of Minecraft for different servers you might be hosting. Yadda yadda yadda, we just want the latest version of Minecraft for our server, which is pretty easy. We have to invoke a magic hack the MSM folks put in and then tell MSM to download the latest version:

```
msm jargroup changeurl minecraft minecraft
msm jargroup getlatest minecraft
```

The last step is to create the server -- just pick whatever name you want.

```
msm server create myserver
```

Here's the biggest and stupidest gotcha of all of this. You need to accept the EULA before you can run your server, but MSM doesn't handle that very elegantly... or at all. The server will not give any indication it failed to start, but running `msm myserver status` will indicate that it's still not running. The fix is simple but inane. Switch to the `minecraft` user and update the `eula.txt` file in the server's directory.

```
sudo su - minecraft
echo eula=true > /opt/msm/servers/myserver/eula.txt
```

Now, start your server.

```
msm myserver start
```

Boom. Now's a great time to test it out, after one more step: make yourself the operator on your server, assuming that's something you want.

```
msm myserver op add your-username
```

### Test It Out

Fire up Minecraft and connect to your new server to make sure everything is working.

![Minecraft Screenshot](http://i.imgur.com/GsmkD5f.png)

# Maps

Now that we've got a self-hosted server up and running, what about those sweet maps?

Those are borne from the power of [The Minecraft Overviewer](http://overviewer.org/), a magic and customizable python script that knows how to render maps and bits of info into a Google Maps-style interface. You can make it do a TON of things, but I'll walk you through a general setup with just a few minor customizations that I use to produce [minecraft.aaronlerch.com](http://minecraft.aaronlerch.com). Check out [the documentation](http://docs.overviewer.org/en/latest/) for all the details of what overviewer can do.

What we're going for is this:

1. configuration to instruct overviewer to render two map layers (Daytime and Caves) and two sets of points of interest (Signs and Players)
2. local web server ([nginx](http://nginx.org/)) for easy testing and local hosting
3. Amazon S3-hosted static website
4. cron job to render maps and sync to AWS S3

### Setup

First, get your prerequisite utilities in place, nginx for a web server and the AWS command line tools.

```
sudo apt-get install nginx
sudo apt-get install awscli
```

### AWS S3 and Security

Always with the security, I know. Hosting a website in S3 is the cheapest option you'll ever find. And the AWS command line tools support for synchronizing a directory into an S3 bucket is hard to beat too. Let's get S3 set up and ready.

Create a new S3 bucket and configure it for static website hosting. I won't cover how to do that here, but [Amazon has some documentation that helps](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html). Just make sure the Index Document for your S3 website is set to `index.html`.

For the purposes of the rest of this, I'm going to assume the name of the bucket you created is `map.example.com`.

On the server we'll need to have valid credentials to access the S3 bucket and upload files to it. We could leverage EC2 IAM Roles for permissions, but for now let's just create a single-purpose IAM user and configure things using this user's credentials.

Create a new IAM user named `minecraft` and make sure "Generate an access key for each user" is checked, we need API access. Capture the Access Key ID and Secret Access Key, those are important.

Edit the user in IAM and under Permissions click "Attach User Policy". Select "Custom Policy", give it a unique name (doesn't matter what), and use the following document, making sure to use the correct bucket name.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1418101359000",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectAcl",
        "s3:GetObjectVersion",
        "s3:ListAllMyBuckets",
        "s3:ListBucket",
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": [
        "arn:aws:s3:::map.example.com"
      ]
    },
    {
      "Sid": "Stmt1418101444000",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectAcl",
        "s3:GetObjectVersion",
        "s3:ListAllMyBuckets",
        "s3:ListBucket",
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": [
        "arn:aws:s3:::map.example.com/*"
      ]
    }
  ]
}
```

This security policy is perhaps a teeny bit overly permissive, but I've found there are few things I dislike more than trying to guess precisely which permissions are needed for various AWS "things". Mostly I'm going for "this user can't cost me tons of money if compromised".

Okay, S3 is ready to go -- let's get generate something to put there.

### Overviewer Configuration

There's a little bit to setting up and configuring overviewer, unfortunately, so stay with me here.

First up, install Overviewer [following their Ubuntu install directions](http://docs.overviewer.org/en/latest/installing/#debian-ubuntu).

```
# Add the custom repo to the list
sudo echo deb http://overviewer.org/debian ./ >> /etc/apt/sources.list

# Get the key for the signed repo
wget -O - http://overviewer.org/debian/overviewer.gpg.asc | sudo apt-key add -

# Update the repo sources and install overviewer
sudo apt-get update
sudo apt-get install minecraft-overviewer
```

Then, create the Overviewer configuration.

```
# Switch to the minecraft user
sudo su - minecraft

# Create a directory to hold the overviewer config
mkdir /opt/msm/.overviewer

# Start with (and customize!) an example overviewer.conf file, or
# create one using the content in the next code block
wget -O /opt/msm/.overviewer/overviewer.conf https://gist.githubusercontent.com/aaronlerch/15369228265cda5bbd3d/raw/114f0e7419af9e798f38cab6cbf0bd1ae6df262b/overviewer.conf
```

```
worlds["myserver"] = "/opt/msm/servers/myserver/world"

def playerIcons(poi):
    if poi['id'] == 'Player':
        poi['icon'] = "http://overviewer.org/avatar/%s" % poi['EntityId']
        return "Last known location for %s" % poi['EntityId']

def signFilter(poi):
    if poi['id'] == 'Sign':
        return "\n".join([poi['Text1'], poi['Text2'], poi['Text3'], poi['Text4']])

renders["normalday"] = {
    "world": "myserver",
    "title": "Daytime",
    "rendermode": smooth_lighting,
    "dimension": "overworld",
    "markers": [dict(name="All signs", filterFunction=signFilter),
                dict(name="All players", filterFunction=playerIcons)],
}

renders["caves"] = {
    "world": "myserver",
    "title": "Caves",
    "rendermode": cave,
    "dimension": "overworld",
    "markers": [dict(name="All players", filterFunction=playerIcons)],
}

# Uncomment the line below if you want to customize the default index.html
# customwebassets = "/opt/msm/.overviewer/customassets"
outputdir = "/usr/share/nginx/html/map"
```

In a nutshell, this is configuring the primary server world to include two render layers, each with two sets of markers, and to put the results into a subdirectory of the default nginx html directory. Speaking of which, we need to create this directory and give the `minecraft` user permissions to it.

```
sudo mkdir -p /usr/share/nginx/html/map
sudo chown minecraft:minecraft /usr/share/nginx/html/map
```

The only thing left to configure Overviewer is to download the map textures, since Overviewer doesn't ship with them, it relies on Minecraft itself for them. The easiest way to do this, if not a bit inefficient, is to install the minecraft client locally. [Overviewer has helpful instructions because this changes a bit over time.](http://docs.overviewer.org/en/latest/running/#installing-the-textures) Using the current latest version, 1.8.1, the following works:

```
sudo su - minecraft
VERSION=1.8.1
wget https://s3.amazonaws.com/Minecraft.Download/versions/${VERSION}/${VERSION}.jar -P ~/.minecraft/versions/${VERSION}/
```

And that's it! (That was a lot.)

Give it a try, it'll take a while to run but not horribly long on a brand new world. Overviewer runs in 2 passes if you are also generating points of interest. Future runs will only render differences from the previous render, so speeds should be faster.

```
overviewer.py --config=/opt/msm/.overviewer/overviewer.conf --genpoi
overviewer.py --config=/opt/msm/.overviewer/overviewer.conf
```

After it runs, because we opened port 80 in the AWS Security Group, we can hit the server directly to see the results: [http://minecraft.example.com/map](http://minecraft.example.com/map).

Success! (Hopefully.)

### Upload to S3

If you are running a server 24/7, you can probably just host the maps from the server itself. But if you, like me, are hosting this casually, it's more fun if the maps are always accessible regardless of the server's availability.

Configure the AWS CLI to use the Access Key ID and Secret Access Key for the IAM user you created above by running `aws configure`. Leave the default region name and default output format empty.

Test the upload/sync is successful with this command (don't forget to use your bucket name)

```
aws s3 sync /usr/share/nginx/html/map/ s3://map.example.com/
```

After it completes successfully, you should be able to visit http://map.example.com/ and see your map!

### cron

The last step is to set up a cron job to automate the map generation and S3 upload process. I have a simple helper script called `update-map.sh` that can give you a starting point.

```
sudo su - minecraft
wget -O /opt/msm/update-map.sh https://gist.githubusercontent.com/aaronlerch/15369228265cda5bbd3d/raw/1915441bac0c8556e9c15d5d829ff946b1764e4b/update-map.sh
chmod 775 /opt/msm/update-map.sh
# Don't forget to update the script to use your bucket name
```

Then configure cron to run this every 20 minutes, or however frequently you want the map updated.

```
sudo su - minecraft
crontab -e
```

```
*/20 * * * * /opt/msm/update-map.sh >/dev/null 2>&1
```

And there you have it, you've got an available minecraft server with automatically updating independently-hosted maps! And, you made it all the way through this mini-guide, gold star for you.

There's plenty of room for customization, and I made some assumptions about the level of sophistication in hosting your Minecraft server and generated maps. Hopefully it's a good starting point, if nothing else. Above all else, have fun!