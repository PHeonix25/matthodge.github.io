---
layout: post
title:  "ChatsOps on Windows with Hubot and PowerShell"
date:   2016-01-17 13:37:00
comments: false
---

ChatOps is a term used to describe conversation driver development or operations for an Ops or Development team. It involves having everyone in the teams in a single chatroom, and brining tools into the chatroom which can help the team to automate, collaborate and work better as a team. The tools or automations are usually exposed using a chat bot which users in the chatroom can talk to to have it take actions, some examples of this may be:

* Have the bot kick off a script
* Check who is on call via the PagerDuty API
* Query a server to see how much disk space is available

Bots can also be a great way to expose functionality to low-privledged users such as help desk staff, without having to create web UI's or forms.

I won't go into any details on the concepts of ChatOps, but I recommend watching **[ChatOps, a Beginners Guide](https://www.youtube.com/watch?v=F8Vfoz7GeHw)** presented by [Jason Hand](https://twitter.com/jasonhand) if you are new to the term.

A popular combination of tools for ChatOps is [Slack](https://slack.com/) for the chat client, and [Hubot](https://hubot.github.com/) as the bot, which is what this post will be targeting. This post will also be using a PowerShell module I've written called [PoshHubot](https://github.com/MattHodge/PoshHubot). The module will handle installation and basic administration Hubot.

* TOC
{:toc}

## Basic Hubot Concepts

There are a few basic Hubot concepts I want to introduce to you before we continue.

### Node.js and CoffeeScript
Hubot is built in CoffeeScript, which is a programming language that complies into JavaScript. Hubot is built on Node.js. This means the server running the bot will need to have Node.js and CoffeeScript installed. The `PoshHubot` module will handle this.

When writing scripts for your bot, you will have to get your hands a little dirty with CoffeeScript. We will be calling PowerShell from inside CoffeeScript, so we only need to know a tiny bit to get by.

### Environment Variables
Hubot and its addons / scripts makes heavy use of environment variables to set certain options for the bot.

One example of this is to allow the Hubot to access sites with invalid SSL certificates, you would set an environment variable of `NODE_TLS_REJECT_UNAUTHORIZED`.

There are 3 possible ways to do this:

* You can set these environment variables as a system wide setting in an Administrative PowerShell prompt using:
{% highlight powershell %}
# This will need to be done with an Administrative PowerShell Prompt
[Environment]::SetEnvironmentVariable("NODE_TLS_REJECT_UNAUTHORIZED", "0", "Machine")
{% endhighlight %}

* You can set them in the current PowerShell instance before you start the bot  using:
{% highlight powershell %}
  $env:NODE_TLS_REJECT_UNAUTHORIZED = '0'
{% endhighlight %}

* You can store the environment variable in the `config.json` file that we generate during the Hubot installation, which the  `PoshHubot` module will load before starting the bot.


### Bot Brain
Hubot has a *brain* which is simply a place to store data you want to persist after Hubot reboots. For example, you could write a script to have Hubot store URL's for certain services, which you could append to via chat commands. You want these URL's to persist after Hubot reboots, so it needs to save them to its brain.

There are many brain adapters for Hubot, for example MySQL, Redis and Azure Blob Storage. For this blog I will be using a file brain - which will just store the brain as a *.json* file on the disk.

## Requirements
You will need to a have a few things ready to get a Hubot setup with Slack:

1. A Windows Machine with PowerShell 4.0+. For this tutorial I will be using a Windows 2012 R2 Standard machine with GUI. Once you get comfortable with Hubot you may decide to switch to using Server Core which is a great choose for running Hubot
1. Administrative access in your Slack group to create a Hubot integration

## Create a Slack Integration for Hubot

To have Hubot speaking to Slack, we need to configure an integration. From Slack:

* Choose **Apps & Custom Integrations**

![Slack - Apps & Custom Integrations](/images/posts/chatops_on_windows/slack_apps_customize.png "Slack - Apps & Custom Integrations")

* Search for **Hubot** and choose **Install**
* Provide a Hubot username - this will be the name of your bot. For this blog the bot will be called *bender*
* Click **Add Hubot Integration**

After the integration has been added, you will be provided an API Token, something like `xoxb-XXXXX-XXXXXX`. We will need this later so note it down.

Additionally, you can customize your bots icon and add channels at this screen.

![Slack - Bot name & Icon](/images/posts/chatops_on_windows/slack_choose_icon.png "Slack - Bot name & Icon")

## Installing Hubot
Install the `PoshHubot` module by downloading it from git and placing it into your PowerShell Module directory.

First we are going to create a configuration file that `PoshHubot` will use.

{% highlight powershell %}
# Import the module
Import-Module -Name PoshHubot -Force

# Create hash of configuration options
$newBot = @{
    Path = "C:\PoshHubot\config.json"
    BotName = 'bender'
    BotPath = 'C:\myhubot'
    BotAdapter = 'slack'
    BotOwner = 'Matt <matt@email.com>'
    BotDescription = 'my awesome bot'
    LogPath = 'C:\PoshHubot\Logs'
    BotDebugLog = $true
}

# Splat the hash to the CmdLet
New-PoshHubotConfiguration @newBot
{% endhighlight %}

Next, we need to install all the required components for Hubot, which will be handled by the `Install-Hubot` command.

{% highlight powershell %}
# Install Hubot
Install-Hubot -ConfigPath 'C:\PoshHubot\config.json'
{% endhighlight %}

This will install the following:

* Chocolatey
* Node.js
* Git
* CoffeeScript
* Hubot Generator
* [Forever](https://github.com/foreverjs/forever) which will run Hubot as a background process.

### Removing Hubot Scripts

Hubot comes installed with some default scripts which are not required when running on Windows. We can use the `Remove-HubotScript` command to remove them.

{% highlight powershell %}
# Be sure to provide the correct ConfigPath for your bot
Remove-HubotScript -Name 'hubot-redis-brain' -ConfigPath 'C:\PoshHubot\config.json'
Remove-HubotScript -Name 'hubot-heroku-keepalive' -ConfigPath 'C:\PoshHubot\config.json'
{% endhighlight %}


### Installing Hubot Scripts

We will now install some third party Hubot scripts using `Install-HubotScript`.

{% highlight powershell %}
# Authentication Script, allowing you to give permissions for users to run certain scripts
Install-HubotScript -Name 'hubot-auth' -ConfigPath 'C:\PoshHubot\config.json'
# Allows reloading Hubot scripts without having to restart Hubot
Install-HubotScript -Name 'hubot-reload-scripts' -ConfigPath 'C:\PoshHubot\config.json'
# Stores the Hubot brain as a file on disk
Install-HuBotScript -Name 'jobot-brain-file' -ConfigPath 'C:\PoshHubot\config.json'
{% endhighlight %}

## Starting Hubot

Before we can start our bot and connect it to slack, we have to configure the environment variables required by the scripts we are using. A good way to find out what environment variables a script is using is to look it up on github. For example,
the [jubot-brain-file](https://github.com/8DTechnologies/jobot-brain-file) script requires `FILE_BRAIN_PATH` to be set.

![jubot-brain-file Script](/images/posts/chatops_on_windows/bot_brain_coffee.png "jubot-brain-file Script")

Additionally, the [Slack adapter](https://github.com/slackhq/hubot-slack) for Hubot requires an environment variable to be set for the Slack API token called `HUBOT_SLACK_TOKEN`.

We will store both of these in the `config.json` file we created earlier.

Open the `C:\PoshHubot\config.json` file and in the `EnvironmentVariables` section, add the new environment variables.

The completed `config.json` file should look something like this:

{% highlight json %}
{
  "Path": "C:\\PoshHubot\\config.json",
  "BotAdapter": "slack",
  "BotDebugLog": {
    "IsPresent": true
  },
  "BotDescription": "my awesome bot",
  "BotPath": "C:\\myhubot",
  "BotOwner": "Matt <matt@email.com>",
  "LogPath": "C:\\PoshHubot\\Logs",
  "BotName": "bender",
  "ArgumentList": "--adapter slack",
  "BotExternalScriptsPath": "C:\\myhubot\\external-scripts.json",
  "PidPath": "C:\\myhubot\\bender.pid",
  "EnvironmentVariables": {
    "HUBOT_ADAPTER": "slack",
    "HUBOT_LOG_LEVEL": "debug",
    "HUBOT_SLACK_TOKEN": "xoxb-XXXXX-XXXXXX",
    "FILE_BRAIN_PATH": "C:\\PoshHubot\\"
  }
}
{% endhighlight %}

With all the configuration in place, we can start Hubot!

{% highlight powershell %}
Start-Hubot -ConfigPath 'C:\PoshHubot\config.json'
{% endhighlight %}

Open up Slack and check if your bot came online! Hubot comes with some built in commands, so you can directly message your bot with `help` and see if you get a response back.

:warning: If for some reason your bot doesn't connect, you can find the logs in the `LogPath` defined earlier in the `config.json` file.

![Speaking to Hubot for the first time](/images/posts/chatops_on_windows/speaking_to_hubot_in_slack.png "Speaking to Hubot for the first time")

If you want your bot to join certain channels, you can enter `/invite @bender` in Slack to bring him into the channel. To have Hubot perform commands, you need to address him in the channel. Try a `@bender pug bomb me`.

![Yay! Pug Bombed!](/images/posts/chatops_on_windows/bot_pug_bomb.png "Yay! Pug Bombed!")

## Integrating Hubot with PowerShell

We have our Hubot joined to Slack and we have triggered a few pug bombs, but it is time to do something useful - create our own script.

The [Hubot documentation](https://hubot.github.com/docs/scripting/) covers scripting in detail and I recommend giving it a read before continuing on.

We are going to write a basic script to find the status of a Windows service on the machine hosting the Hubot. The plan is:

* Send the bot a message saying `@bender: get service dhcp` - where `dhcp` could by any string.
* The Hubot script will use a capture group to select out the name of the service (in this case `DCHP`)
* The Hubot script will pass this captured service name into a PowerShell script to find the status of the service.
  * If the service exists, it will return the status.
  * If the service does not exist, it will say the service does not exist.
* The PowerShell script will return the results in a json format. This will make it far easier to work with in CoffeeScript

### Install Edge.js and Edge-PS

[Edge.js](https://github.com/tjanczuk/edge) and [Edge-PS](https://github.com/dfinke/edge-ps) are Node.js script which allow calling .NET and PowerShell (among other things) from Node.js.

To use them inside Hubot, we need to add them to `package.json` file which is generated when we install Hubot for the first time. You can find `package.json` in the `BotPath` specified above, in our case it is `C:\myhubot\packages.json`. We will also add a version constraint. The latest version of each package can be found by searching the [npm package manager](https://www.npmjs.com).

After you have added them your `package.json` should look similar to this:

{% highlight json %}
{
  "name": "bender",
  "version": "0.0.0",
  "private": true,
  "author": "PoshHubot <posh@hubot.com>",
  "description": "PoshHubot is awesome.",
  "dependencies": {
    "hubot": "^2.18.0",
    "hubot-diagnostics": "0.0.1",
    "hubot-google-images": "^0.2.6",
    "hubot-google-translate": "^0.2.0",
    "hubot-help": "^0.1.3",
    "hubot-heroku-keepalive": "^1.0.2",
    "hubot-maps": "0.0.2",
    "hubot-pugme": "^0.1.0",
    "hubot-redis-brain": "0.0.3",
    "hubot-rules": "^0.1.1",
    "hubot-scripts": "^2.16.2",
    "hubot-shipit": "^0.2.0",
    "hubot-slack": "^3.4.2",
    "edge": "^5.0.0",
    "edge-ps": "^0.1.0-pre"
  },
  "engines": {
    "node": "0.10.x"
  }
}
{% endhighlight %}

### Create the PowerShell Script

We need to design a script that can be called from CoffeeScript, the Hubot scripting language. I have some standard methods of creating PowerShell scripts that will be called from Hubot to make things easier.

* **Create functions with paramaters** - All advanced scripts should be functions, but it just makes it nice and easy to call from CoffeeScript when they have well defined parameters
* **Put error handling in your script** - Use try-catch blocks inside your PowerShell functions so you can return a message to the bot if the command has failed
* **Always output in json** - This is a far easier way to pass data back to CoffeeScript and means you can use PowerShell objects to send data back and have CoffeeScript pick out the parts you want

With this methods in mind, here is the function I came up with to find the service status:

{% gist 66bf00bedb98d72c2506 %}

I am applying some [Slack formatting](https://get.slack.help/hc/en-us/articles/202288908-Formatting-your-messages) in my output, including the use of asterisks around words for bold and back ticks for code blocks. You will notice there are double backticks in the code so PowerShell does not interpret them.

Here is some example output from the PowerShell when the function is run against a service that exists:

{% highlight powershell %}
# Dot Source the function
. .\Get-ServiceHubot.ps1

# Get a service that exists on the system
Get-ServiceHubot -Name dhcp
{% endhighlight %}

{% highlight json %}
{
    "success":  true,
    "output":  "Service dhcp (*DHCP Client*) is currently `Running`"
}
{% endhighlight %}

Here is some example output from the PowerShell when the function is run against a service that doesn't exist on the machine:

{% highlight powershell %}
# Dot Source the function
. .\Get-ServiceHubot.ps1

# Get a service that exists on the system
Get-ServiceHubot -Name MyFakeService
{% endhighlight %}

{% highlight json %}
{
    "success":  false,
    "output":  "Service MyFakeService does not exist on this server."
}
{% endhighlight %}