{% include '/version.md' %}

# Facts

## Quest objectives
 - TBD

## Getting started

The class parameters you learned about in the previous quest let you reduce
many of the tasks involved in configuring an application (or whatever system
component your module manages) to a single, well-defined interface. You can
make your life even easier by writing a module that will automatically use
available system data to set some of its variables.

In this quest, we'll discuss `facter`. The
`facter` tool allows Puppet to automatically access information about the
system where an agent is running as variables within your Puppet manifest.

As you'll see in this quest, these facts can be quite useful on your own. In
the next quest, you will see how they can be used along with **conditional
statements** to let you write Puppet code that will behave differently in
different contexts.

To start this quest enter the following command:

    quest begin facts

## Facter

>Get your facts first, then distort them as you please.

> -Mark Twain

You already encountered `facter` tool when we asked you to run `facter
ipaddress` in the setup section of this Quest Guide. We briefly discussed
the tool's role in a Puppet run—the Puppet agent runs `facter` to get a list
of facts about the system to send to the Puppet master as it requests a
catalog. The Puppet master then uses these facts to help compile that catalog
before sending it back to the Puppet agent to be applied.

Before we get into integrating facts into your Puppet code, let's use the
`facter` tool from the command line to see what kinds of facts are available
and how they're structured.

First, go connect to the agent node we've set up for this quest.

    ssh learning@pasture01.puppet.vm

You can access a standard set of facts with the `facter` command. Adding the
`-p` flag will include any custom facts that may you may have installed on the
Puppet master and synchronized with the agent during the pluginsync step of a
Puppet run. We'll pass this `facter -p` command to `less` so you can scroll
through the output in your terminal.
	
    facter -p | less

When you're done, press `q` to quit `less`.

Notice that the output of this command is shown as a hash. Some facts, such as
`os` include data in a nested JSON format.

    facter -p os

You can drill down into this structure by using dots (`.`) to specify the key
at each child level of the hash, for example:

    facter -p os.family

Now that you know how to check what data are available via `facter` and how the
data are structured, let's return to the Puppet master so you can see how this
can be integrated into your Puppet code.

    exit

All facts are automatically made available within your manifests. You can
access the value of any fact via the `$facts` hash, following the
`$facts['fact_name']` syntax. To access structured facts, you can chain more
names using the same bracket indexing syntax. For example, the `os.family` fact
you accessed above is available within a manifest as `$facts['os']['family']`.

Let's take a break from the Pasture module you've been working on. Instead,
we'll create a new module to manage an MOTD (Message of the Day) file. This
file is commonly used on *nix systems to display information about a host when
a user connects. Using facts will allow you to create a dynamic MOTD that can
display some basic information about the system.

Creating a new module will also help review some of the concepts you've learned
so far.

From your `modules` directory, create the directory structure for a module
called `motd`. You'll need two subdirectories called `manifests` and
`templates`.

    mkdir -p motd/{manifests,templates}

Begin by creating an `init.pp` manifest to contain the main `motd` class.

    vim motd/manifests/init.pp

This class will consist of a single `file` resource to manage the `/etc/motd`
file. We'll use a template to set the value for this resource's `content`
parameter.

```puppet
class motd {

  $motd_hash = {
    'fqdn'       => $facts['networking']['fqdn'],
    'os_family'  => $facts['os']['family'],
    'os_name'    => $facts['os']['name'],
    'os_release' => $facts['os']['release']['full'],
  }

  file { '/etc/motd':
    content => epp('motd/motd.epp', $motd_hash),
  }

}
```

NOTE: The `$facts` hash and top-level (unstructured) facts are automatically
loaded as variables into any template. To improve readibility and reliability,
we strongly suggest suggest using the method shown here. Just be aware that you
might encounter templates in the wild with direct references to facts.

Now we'll create the `motd.epp` template.

    vim motd/templates/motd.epp

We'll begin with a parameters tag to make the set of variable we're using
explicit. Note that the MOTD is a plaintext file without any commenting syntax,
so we'll leave out the conventional "managed by Puppet" note.

```
<%- | $fqdn,
      $os_family,
      $os_name,
      $os_release,
| -%>
```

Next, we'll add a simple welcome message and use our Puppet facts to provide
some basic information about the system.

```
<%- | $fqdn,
      $os_family,
      $os_name,
      $os_release,
| -%>
Welcome to <%= $fqdn %>

This is a <%= $os_family %> system running <%= $os_name %> <%= os_release %>

```

With this template set, your simple MOTD module is complete. Open your
`site.pp` manifest to assign it to the `pasture.puppet.vm` node. 

    vim /etc/puppetlabs/code/environments/production/manifests/site.pp

We're not using any parameters, so we'll use the `include` function to add the
`motd` class to the `pasture.puppet.vm` node definition:

```puppet
node 'pasture.puppet.vm` {
  include motd
  class { 'pasture':
    default_character => 'cow',
  }
}
```

Once this is complete, connect again to the `pasture.puppet.vm` node.

    ssh learning@pasture.puppet.vm

And trigger a Puppet agent run:

    puppet agent -t

To see the MOTD, first, disconnect from `pasture.puppet.vm`.

    exit

Now reconnect.

    ssh learning@pasture.puppet.vm

## Review

TBD