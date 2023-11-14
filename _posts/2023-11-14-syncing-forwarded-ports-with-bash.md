---
title: Syncing Forwarded Ports with Bash
date: 2023-11-14 14:11 -0500
category: homelab
tags: [automation, homelab, bash, scripting, guide]
---
In this post I'd like to explain how I solved a simple problem in Linux with bash scripting. This was actually the first project where I made use of bash scripts and I had to learn a lot along the way. The problem was this: I use a VPN service (Private Internet Access, or PIA) that offers port forwarding which I use with a bittorrent client (qBittorrent, or qBit). They work well together, but occasionally the forwarded port in PIA would change and need to be manually updated in qBittorrent's settings. Let's talk about how to automate this!

## Install CLI tool and read documentation

Our first step is to [install the command line interface for qBittorrent](https://github.com/fedarovich/qbittorrent-cli). PIA's desktop client already includes a CLI, so we won't need to do anything there but refer to the existing docs for available commands and options. I'll provide links to the CLI documentation for both PIA and QBT here:

- [Private Internet Access CLI](https://helpdesk.privateinternetaccess.com/kb/articles/pia-desktop-command-line-interface-2)
- [qBittorrent CLI](https://github.com/fedarovich/qbittorrent-cli/wiki/command-reference)

While the documentation for both services cover a lot of possible actions and settings, we're really just interested in a few key commands to configure port forwarding. In PIA, we're looking for a command to show us the currently forwarded port. The appropriately-named command `get` is exactly what we're looking for, paired with the `portforward` type. The full command, `piactl get portforward`, returns the current port number or a status indicator. The possible statuses and what they indicate are as follows:

- Attempting        : PIA is still connecting
- Inactive          : PIA is disconnected or port forwarding is disabled
- Unavailable       : port forwarding is not available; similar to 'Failed'
- Failed            : PIA failed to assign a port number

We'll deal with handling these statuses later on, but for now just be aware that they exist as possible values our command may return. In qBittorrent, we're looking for a command to change the existing port assignment. You can find it not-so-intuitively tucked away under `server`>`settings`>`connection` in the command reference. The full command, `qbt server settings connection -p <PORT>`, sets the port in the client, exactly as if we had done so manually in the program's settings panel. Take note that running this same command without the port flag and port number will display some really useful information about our QBT server's connection---more on this in a second. This would also be a great time to test and practice these commands in a terminal to make sure they work as expected.

## Combine and craft commands

Now that we've figured out the commands, all we need to do is combine them to accomplish our goal of setting the forwarded port automatically when it changes. To start, let's look at the output from the `qbt server settings connection` command:

![img of qbt connection output](/assets/img/qbt-connection-output.jpg)

You can see this has printed out some info about our qBittorrent connection, including the port that's assigned for incoming connections. We'll need this information for later, but in order to make use of it in our automation script, we need to isolate the port number and omit everything else. There are a few options that come to mind for picking out this data; my first attempt was to use `awk` since the command output appears to be in a table format. This turned out to be a red-herring, as the output actually isn't formatted correctly for `awk` to pick out the columns easily. Instead, we'll use `grep` to pick out the whole line:

`qbt server settings connection | grep -Po 'Incoming connections port:\s+[0-9]+'`

This took me a bit of trial-and-error to figure out, as well as a [regex calculator](https://regex-generator.olafneumann.org/) to help me craft the right regular expression to pick out the port number. If you've never used `grep` or `awk`, be sure to check their man pages and other documentation/resources for more information about how they work. Let's talk through this command, bit by bit.

First we call our command to list the connection info, followed by a pipe (` | `). Pipes are used to direct the output of one command into the following command, which in this case is `grep`. We're essentially telling `grep` to use the previous command's output as its input. `grep` is often used to search and filter text using a specific file as input, but it will also accept the piped output of another command, rather than a file. Actually, many Linux CLI programs can be piped together like this! The option `-P` tells it to use Perl syntax (needed for the regular expression to work) and `-o` is shorthand for `--only-matching` which tells it to only output matching information (needed to omit the unneeded info from our connection output). Lastly, the regular expression `'Incoming connections port:\s+[0-9]+'`, which starts with the exact text string where our line begins ("Incoming connections port:"), followed by `\s+` which is a regex pattern to match all the whitespace in between the string and our port value, then finally `[0-9]+`, the regex pattern to match that port value. Importantly, all of this is wrapped in single quotes, which ensures only this line is selected and the preceding and following whitespace and other lines aren't printed in our final output. Thank goodness for regex calculators, right? Let's check what this command outputs:

{% highlight console %}
$ qbt server settings connection | grep -Po 'Incoming connections port:\s+[0-9]+'
Incoming connections port:                  34841
{% endhighlight %}

Now that we've isolated this to a single line, we can use `awk` to grab just the port number at the end of our line. This turns out to be a very simple command: `awk '{ print $NF}'`, which tells awk to print the last field of a record. Our record is only one line and the last record is the port value at the end of that line. We can add this command right on the end of our `grep` command from above, separated by another pipe to tell `awk` to use the output from the command before it. That leaves us with `qbt server settings connection | grep -Po 'Incoming connections port:\s+[0-9]+' | awk '{ print $NF}'` which returns the incoming port number that is currently set in qBittorrent. Save this command for now and we'll return to it later.

At this point we've figured out all the individual components needed to find and set the port, so we can actually craft a one-line command that does what we're looking for:

`qbt server settings connection -p $( piactl get portforward )`

`$( )` is a command substitution and it allows us to substitute the output of one command inside another. One problem with this one-liner is that `piactl get portforward` can also output status information as a string, so we also need a way to handle that. Additionally, this one-liner needs to be invoked manually. We can solve both of these problems by turning our command into a bash script.

## Intro to Bash scripting

So, what the heck is a bash script? In Linux, the terminal interface uses any number of 'shells', Bash ([Bourne-Again SHell](https://tiswww.case.edu/php/chet/bash/bashtop.html)) being one of them. Most shells, including Bash, are both command line interpreters and scripting language interpreters. Many shells support '[shell scripting](https://www.shellscript.sh)', which allow commands to be piped together in series and manipulated with common programming concepts like arithmetic, variables, flow control and other logic tests, and so on. The commands you run on a shell's command line can be run as a script, and a script can also be run right on the command line for the shell to interpret. However, it's much more practical to save scripts as a file to be executed later, which is exactly what we're going to do.

First, let's outline some objectives to keep us on track while we work on our script:

- check and compare current ports
- handle all possible PIA port status strings
- set the port in QBT

With these in mind, let's move on to the script. To begin, `touch script_name.sh` to create our file and use your text editor of choice to begin editing. We'll start with a shebang (`#!`), which tells our interpreter that this file is a script and what shell to use. It's also a good idea to add comments for yourself and others to explain what is going on throughout.

{% highlight console %}
#!/bin/bash
#This script compares the currently forwarded port in PIA VPN with the assigned port in qBittorrent
#If matching, do nothing; if different (and depending on PIA status), pass the updated port to qbt
{% endhighlight %}

### Variables

The first thing we want this script to do is check our ports so we can compare them later, and it would be a good idea to store these values somewhere, too. That's exactly what variables are for, and we can save the result of our connection test command from earlier as a variable like so:

{% highlight console %}
#store current ports as variables
QBT=$( qbt server settings connection | grep -Po 'Incoming connections port:\s+[0-9]+' | awk '{ print $NF}' )
PIA=$( piactl get portforward )
{% endhighlight %}

We can call our variables to grab these stored values with `$VAR`, where 'VAR' is our variable name. Let's write a command to print out the port assignments using `echo`. This will be helpful for debugging---if anything goes wrong, our logs will display the result of our earlier stored commands and we can spot any problems.

{% highlight console %}
echo "qbt port: $QBT   pia port: $PIA"
{% endhighlight %}

### Tests and logic

Now we need to think about our logic. We'll need to structure some `if` statements (or 'tests') to tell our script what to do when the ports match, what to do when they don't match, and what to do when PIA give us a port status message. Easiest first: when the port values match, nothing should change. Something like `if [ "$PIA" = "$QBT" ]; then echo "qBittorrent port is current; else echo "Unexpected error"; fi` will print out a success message but otherwise take no action with the ports. Note that we enclosed our variable in quotation marks. This syntax is required for the variable values to be compared correctly in our test statement. The `else` statement is for later, and will serve as a generic error message if none of our other logic works. `fi` closes our `if` statement, completing the logic test. We'll save this command for later but will need to modify it to work with the rest of our script as it comes together.

With the logic done for when ports match, let's move on to when things don't match. An `if` statement that checks if our ports match will look like this:

{% highlight console %}
if [ "$PIA" != "$QBT" ]
#if ports don't match
then
  echo "Ports do not match ..."
{% endhighlight %}

Just like our last statement, make sure to enclose variables in quotes when comparing them. Adding an exclamation mark (`!`) in front of our equals operator (`=`) turns it into a 'does not equal' operator. We've also added an `echo` to print a message---we'll continue to add messages and comments like this as we complete our script. After we've compared our ports and found them to be not-matching, we'll need to think about what to do next. Our PIA command from earlier has several outputs, so we'll need an `if` statement that handles our PIA status messages as well. Our next bit of logic only applies if the ports don't match, so we should nest it within the `if` statement before we close it with `fi`. Indentation isn't required but it will make it a lot easier to read when nesting statements!

{% highlight console %}
if [ "$PIA" != "$QBT" ]
#if ports don't match
then
  echo "Ports do not match ..."
  
  if [ "$PIA" = "Attempting" ]
  #when non-matching, if the status returns as Attempting, prompt user to wait for connection
  then
    echo "Wait for VPN to finish connecting ..."
{% endhighlight %}

If the ports don't match then the command will `echo` our message and then move on to the second `if` statement within, which will test if our reported PIA port status matches the string "Attempting". This status message indicates that PIA is still trying to connect, so all we need to do in this case is wait a bit before running our test script again. We'll need another `if` statement for our other statuses, and we can list them right after the first using `elif`, which means "else if"---this is like saying **if** this first thing happens, then do this; or **else if** this other thing happens, then do this instead. Using `elif`, we can specify different tests and actions to take. Here's how our chunk of code from before looks when we add our next test statement:

{% highlight console %}
if [ "$PIA" != "$QBT" ]
#if ports don't match
then
  echo "Ports do not match ..."
  
  if [ "$PIA" == "Attempting" ]
  #when non-matching, if the status returns as Attempting, prompt user to wait for connection
  then
    echo "Wait for VPN to finish connecting ..."
  
  elif [ "$PIA" == "Inactive" ]
  #if status returns as Inactive, attempt to connect the VPN
  then
    echo "Starting PIA ..."
    piactl connect && sleep 5 ; PIA2=$( piactl get portforward ) && qbt server settings connection -p $PIA2
{% endhighlight %}

Our second test condition is checking our PIA variable against the string "Inactive", a status which means the PIA client is not connected. If it matches, we then `echo` a message and run a command. So far we've only used `piactl` to check the current port, but it can also be used to connect and disconnect the client. The command following our `echo` tells PIA to connect and then pause the terminal for 5 seconds (`piactl connect && sleep 5;`), then attempts to save the newly-generated port number into a second variable (`PIA2=$( piactl get portforward`) before setting it in QBT (`&& qbt server settings connection -p $PIA2`). A few notes here: I saved the port number as a second variable here, but that's probably overkill; `;` and `&&` are used here to chain two or more commands together. When `;` is used, the command before it runs and the next command runs when the first has finished, whether successful or not. When `&&` is used, the first command runs and the next command will only run if the first completes without errors. In this case, we only want our `qbt` command to run if the `piactl` command succeeds.

Next, we need to add a test for our remaining PIA status conditions, "Unavailable" and "Failed". The documentation from PIA on the CLI status messages is pretty sparse, but from what I could tell, both seem to happen when the client attempts to switch to a new port and cannot. I don't know the exact distinction between the two statuses, but they can both be solved by reconnecting, so that's what we'll do. I'll once again share the whole chunk of code to preserve the indentation and nesting for clarity.

{% highlight console %}
if [ "$PIA" != "$QBT" ]
#if ports don't match
then
  echo "Ports do not match ..."
  
  if [ "$PIA" == "Attempting" ]
  #when non-matching, if the status returns as Attempting, prompt user to wait for connection
  then
    echo "Wait for VPN to finish connecting ..."
  
  elif [ "$PIA" == "Inactive" ]
  #if status returns as Inactive, attempt to connect the VPN
  then
    echo "Starting PIA ..."
    piactl connect && sleep 5 ; PIA2=$( piactl get portforward ) && qbt server settings connection -p $PIA2
  
  elif [ "$PIA" == "Unavailable" ] || [ "$PIA" == "Failed" ]
  #if the status returns Unavailable/Failed, disconnect and reconnect PIA
  #+ this generates a new forwarded port which is stored and then passed into qbt
  then
    echo "Restarting PIA and setting new port ..."
    piactl disconnect
    wait
    piactl connect && sleep 5 ; PIA2=$( piactl get portforward ) && qbt server settings connection -p $PIA2
{% endhighlight %}

Just as before, this portion of the script tests for the strings "Unavailable" or "Failed". If matching, it prints a message, then sends `piactl disconnect` and tells the console to `wait` until the previous command exits. Once complete, it then sends the exact same `pia connect` command as our last example. The main difference here is that our status conditions exist while the connection is active, so we need to tell PIA to disconnect before trying to connect again. With this last status taken care of, all that's left is to take an action in the case of a mismatched port **without** a status condition. This one should be pretty easy to figure out:

{% highlight console %}
else
  #if no status was reported but the ports still mismatch, update qbt with the current PIA port
    echo "Setting new forwarded port in qBittorrent ..."
    qbt server settings connection -p $PIA
  fi
{% endhighlight %}

I've shown just an excerpt here, but I'll have an example of the full script in just a moment. The `else` section here should be indented and nested at the same depth as our second `if` and `elif` statements, and will close up the nested section. We print out a message, then run our original `qbt server settings connection -p $PIA` command to set our qBittorrent port to the current Private Internet Access forwarded port. `fi` closes the `if` statement. The last thing we need is to mash our two commands together. Earlier, we made a simple command that tested if our ports matched and printed some messages:

```if [ "$PIA" = "$QBT" ]; then echo "qBittorrent port is current; else echo "Unexpected error"; fi```

Let's adapt this to fit into the rest of our script, which looks like this so far:

{% highlight console %}
#!/bin/bash
#This script compares the currently forwarded port in PIA VPN with the assigned port in qBittorrent
#If matching, do nothing; if different (and depending on PIA status), pass the updated port to qbt

#store current ports as variables
PIA=$( piactl get portforward )
QBT=$( qbt server settings connection | grep -Po 'Incoming connections port:\s+[0-9]+' | awk '{ print $NF}' )

#Debugging- start with var outputs
echo "qbt port: $QBT   pia port: $PIA"


#LOGIC
if [ "$PIA" != "$QBT" ]
#if ports don't match
then
  echo "Ports do not match ..."

  if [ "$PIA" == "Attempting" ]
  #when non-matching, if the status returns as Attempting, prompt user to wait for connection
  then
    echo "Wait for VPN to finish connecting ..."

  elif [ "$PIA" == "Inactive" ]
  #if status returns as Inactive, attempt to connect the VPN
  then
    echo "Starting PIA ..."
    piactl connect && sleep 5 ; PIA2=$( piactl get portforward ) && qbt server settings connection -p $PIA2

  elif [ "$PIA" == "Unavailable" ] || [ "$PIA" == "Failed" ]
  #if the status returns Unavailable/Failed, disconnect and reconnect PIA
  #+ this generates a new forwarded port which is stored and then passed into qbt
  then
    echo "Restarting PIA and setting new port ..."
    piactl disconnect
    wait
    piactl connect && sleep 5 ; PIA2=$( piactl get portforward ) && qbt server settings connection -p $PIA2
  
  else
  #if no status was reported but the ports still mismatch, update qbt with the current PIA port
    echo "Setting new forwarded port in qBittorrent ..."
    qbt server settings connection -p $PIA
  fi

{% endhighlight %}

We've closed our nested `if` statement, but our original `if` is still open and will need to be closed with a final `fi`. First, let's change our earlier test command into an `elif`, and our "Unexpected error" message into an `else`.

{% highlight console %}
elif [ "$PIA" == "$QBT" ]
#if the ports did match, report no change; generic echo for debugging uncaught scenarios
then
  echo "qBittorrent port is current!"
else
  echo "Unexpected error!!"
fi
  {% endhighlight %}

This portion of code can then be placed right after our closed `if` statement, at the same indentation as the very first `if` statement (which is to say, not indented). Here's the whole script:

{% highlight console %}
#!/bin/bash
#This script compares the currently forwarded port in PIA VPN with the assigned port in qBittorrent
#If matching, do nothing; if different (and depending on PIA status), pass the updated port to qbt

#store current ports as variables
PIA=$( piactl get portforward )
QBT=$( qbt server settings connection | grep -Po 'Incoming connections port:\s+[0-9]+' | awk '{ print $NF}' )

#Debugging- start with var outputs
echo "qbt port: $QBT   pia port: $PIA"

#LOGIC
if [ "$PIA" != "$QBT" ]
#if ports don't match
then
  echo "Ports do not match ..."

  if [ "$PIA" == "Attempting" ]
  #when non-matching, if the status returns as Attempting, prompt user to wait for connection
  then
    echo "Wait for VPN to finish connecting ..."

  elif [ "$PIA" == "Inactive" ]
  #if status returns as Inactive, attempt to connect the VPN
  then
    echo "Starting PIA ..."
    piactl connect && sleep 5 ; PIA2=$( piactl get portforward ) && qbt server settings connection -p $PIA2		#there is room here to improve error handling; what if portforward gives a status here?

  elif [ "$PIA" == "Unavailable" ] || [ "$PIA" == "Failed" ]
  #if the status returns Unavailable/Failed, disconnect and reconnect PIA
  #+ this generates a new forwarded port which is stored and then passed into qbt
  then
    echo "Restarting PIA and setting new port ..."
    piactl disconnect
    wait
    piactl connect && sleep 5 ; PIA2=$( piactl get portforward ) && qbt server settings connection -p $PIA2
  
  else
  #if no status was reported but the ports still mismatch, update qbt with the current PIA port
    echo "Setting new forwarded port in qBittorrent ..."
    qbt server settings connection -p $PIA
  fi

elif [ "$PIA" == "$QBT" ]
#if the ports did match, report no change; generic echo for debugging uncaught scenarios
then
  echo "qBittorrent port is current!"
else
  echo "Unexpected error!!"
fi
{% endhighlight %}

### Filename and permissions

We'll want to save this script with a descriptive name and an .sh file extension. I store my scripts in `.local/bin` in my home directory, so my file path is `/home/slept/.local/bin/pia-qbt-ports.sh`. Change the permissions to make it executable with `chmod 755 /path/to/script.sh`. Lastly, we'll want to run this automatically, so let's add this script to our crontab.

### Scheduling with Crontab

So, what the heck is a crontab? It's a table which contains a list of tasks (which could be scripts or commands) to run on a regular schedule. To edit this list, run `crontab -e` in a terminal, add a new line starting with a [cron schedule expression](https://crontab.guru) and ending with the path to our script (or command). `* * * * * /absolute/path/to/script.sh` if you want it to run every minute. You can also [redirect the console output](https://catonmat.net/bash-one-liners-explained-part-three) to specific log files by adding `>/tmp/pia_qbt_ports.log 2>/tmp/pia_qbt_ports.err` after the script path in your crontab. `>` is shorthand for `1>` where `1` represents stdout, the `>` symbol instructs the terminal to redirect output into a new file, overwriting if the file already exists, and `2` represents stderr. Make sure you use the absolute path to your script in crontab, and if you run into any unusual problems with your variable calls (this is where your debug messages will come in handy), you may need to add the folder where your script is stored to PATH in crontab or your script or both (`PATH=/home/slept/.local/bin`). Save your crontab and exit when you're done, then get ready to check your logs for problems or errors. You can disconnect/reconnect PIA to make it generate a new port number and if all goes according to plan, you should see your messages printed to the log files as the ports are changed for you automatically.

That's it! I know it seems like a lot of work to save a few button clicks, but now we'll never have to worry about checking if things are working or finding that torrents have been stalled for who knows how long. Small problems like this are a great opportunity to practice bash scripting and learn more about Linux in general.

Thanks for following along---until next time!
