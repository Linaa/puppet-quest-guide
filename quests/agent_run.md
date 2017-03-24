{% include '/version.md' %}

# Agent run

## Quest objectives

- Understand the process of a Puppet agent run.
- List and sign agent certificates.
- Use the **site.pp** manifest to classify a node.

## Get started

> Quotation

> - Sayer of quotation

The `puppet resource` command you explored in the previous quest let you see a
system the way Puppet does—through the resource abstraction layer. While
exploring and manipulating resources through Puppet's command-line tools is a
useful exercise, the real value of the resource abstraction layer is to serve
as a foundation for the centralization and automation of control over your
infrastructure.

Because the resource abstraction layer provides a universal way to model system
state, it lays the necessary groundwork for communication between a central
Puppet master server and all the agent nodes in the infrastructure you want
Puppet to manage.

In this quest, we'll walk through a Puppet agent run to demonstrate how a
Puppet agent node communicates with the Puppet master server.

Ready to get started? Run the following command on your Learning VM:

    quest begin agent_run

## The master/agent architecture

As we mentioned briefly in the previous quest, Puppet is typically used in
what's called an agent/master (client/server) architecture.

In this architecture, each managed node in your infrastructure runs the
**Puppet agent** application. One or more servers (depending on the size and
complexity of your infrastructure) act as **Puppet master(s)**, and run the
Puppet Server application to handle communication with your agent nodes. (The
Puppet master server will also typically host several supporting services such
as the PE console services and PuppetDB, though in larger deployments these may
be distributed across other servers.)

In this agent/master architecture, the Puppet agent and Puppet master each
play a distinct part in managing your infrastructure.

By default, the Puppet agent runs as a service on your Agent node and initiates
a Puppet run every half-hour. (Note that we've disabled these automatic runs on
the Learning VM's agent nodes. Instead, you'll trigger runs manually to get
more control and visibility as you learn how Puppet works.)

![image](../assets/SimpleDataFlow.png)

The Puppet agent begins a Puppet run by sending a **catalog request** to the
Puppet master. This request includes some information about the agent system
provided by a tool called **facter**. (We'll cover facter in detail in a later
quest.)

Your Puppet master keeps a copy of the Puppet codebase you've created to define
the desired state for systems in your infrastructure. This Puppet code is
primarily made up of resource declarations like the ones you saw in the
previous quest. While your Puppet code's final purpose is to define resources,
it also includes some language features such as variables, classes, and
conditionals that can give you control over which resources you want on a
system and how their parameters are set. The master parses this code to create
a **catalog**. The catalog is the final list of system resources that
define the desired state for an Agent node.

The Puppet master sends this catalog back to the Puppet agent, which then uses
its **providers** to check if the desired state of each resource defined in the
catalog matches the actual state of the resource on the system. If any
differences are found, the providers will make whatever changes are necessary
to bring the actual state of the system into line with the desired state
defined in the catalog.

Finally the Puppet agent generates a report including information about
unchanged resources, successful changes, and any errors it may have encountered
during the run. It sends this report back to the Puppet master, which stores
it in PuppetDB and makes it available via the PE console's web GUI.

## Certificates

There's one more thing to note before we can move on to demonstrating a Puppet
agent run.

All communications between an agent and master happen over SSL. Before the
master will communicate with an agent needs a way to validate that the agent
node itself is authentic. This prevents unauthorized connections from spoofing
an agent node to access potentially sensitive data that might be included in a
catalog. While Puppet does provide [options for encrypting
data](https://puppet.com/blog/encrypt-your-data-using-hiera-eyaml) within a
catalog, it's best to prevent any possibility of access in the first place.

Puppet addresses this by requiring any node contacting the Puppet master to
authenticate with a signed certificate. The first time a Puppet agent contacts
the Puppet master, it will submit a **certificate signing request** (CSR). A
Puppet administrator can then validate that the system sending the CSR should
be allowed to request catalogs from the master before deciding to sign the
certificate. (You can read more about the details of Puppet's cryptographic
security and the certification system on the [docs
page](https://docs.puppet.com/background/ssl/index.html))

<div class = "lvm-task-number"><p>Task 12:</p></div>

Before getting started, go ahead and connect to your agent node. When you began
this quest, the node you used in the last quest was destroyed and a new one was
created. This new node has the Puppet agent pre-installed, so there's no need to
repeat the installation process. Go ahead and connect to the new agent node
with the same credentials you used in the last quest.

**username: learning**  
**password: puppet**

    ssh learning@learning.puppet.vm

First, try to trigger a Puppet agent run without your agent node's certificate
signed. The agent will attempt to contact the Puppet master, but its request
will be rejected.

    sudo puppet agent -t

You'll see a notification like the following:

    Exiting; no certificate found and waitforcert is disabled

No problem, you just have to sign the certificate. For now, we'll show you how
to do it from the command line. If you prefer a GUI, the PE console includes
[tools for certificate
management](https://docs.puppet.com/pe/latest/console_cert_mgmt.html).

<div class = "lvm-task-number"><p>Task 13:</p></div>

First, exit your SSH session to return to the your Puppet master node.

    exit

Use the `puppet cert` tool to list unsigned certificates.

    puppet cert list

Sign the cert for `hello.puppet.vm`.

    puppet cert sign hello.puppet.vm

<div class = "lvm-task-number"><p>Task 14:</p></div>

With that taken care of, your Puppet agent is authorized to make catalog
requests.

## Triggering a Puppet run

As noted above, the default for the Puppet agent service is to initiate a
Puppet run every thirty minutes. Because it would be hard to demonstrate Puppet
clearly with these scheduled background runs we've disabled the Puppet agent
service on your agent node. Instead, you can use the `puppet agent -t` command
to trigger a run manually.

Go ahead and connect to your agent node:

    ssh learning@learning.puppet.vm

Trigger an agent run. Now that the agent's certificate is signed, it will
receive a catalog from the Puppet master.

    sudo puppet agent -t

While you haven't yet told Puppet to manage any resources on the system, you'll
see a lot of text scroll by. Most of what you see is a process is called
[pluginsync]. During pluginsync, any extensions installed on the master (such
as custom facts, resource types, or providers) are synced to the Puppet agent
before the Puppet run continues. This ensures that the agent has all the tools
it needs to correctly apply the catalog.

This pluginsync process adds a lot of clutter, but we'll focus on three lines
that look like the following.

```
Info: Loading facts
Info: Caching catalog for hello.puppet.vm
Info: Applying configuration version '1464919481'
```

This output shows you one side of the conversation between the agent and master
we discussed at the beginning of this quest.

You can see that the Puppet agent loads the facts it needs to share the details
of the system where its running with the Puppet master.

Next, though this output from the agent run doesn't tell you explicitly that it
has received a catalog from the master, you can see when it has because it lets
you know as it caches a copy of this new catalog. (The Puppet agent can be
configured to fail over to this cached catalog if it is unable to connect to
the master.)

Finally, the Puppet agent applies the catalog. Normally, you would see a list
of all the changes made by the agent shown after this step. In this case,
however, the Puppet master didn't find any Puppet code to apply to your agent
node, and didn't make any changes (other than those involved in pluginsync)
during this run.

## Classification

To make something more interesting happen, you'll have to specify a desired
state for some resources on the `hello.puppet.vm` node.

<div class = "lvm-task-number"><p>Task 15:</p></div>

Remember, the Puppet code you use to describe how you want a node to be
configured lives on the Puppet master. End your SSH session on the
`hello.puppet.vm` agent node to return to the Puppet master:

    exit

Before diving in and writing some Puppet code, let's take a moment to go over
the catalog compilation process from the Puppet master's perspective. This will
help you understand exactly what you're doing as you write code to apply to
your agent.

When the Puppet Server application on the Puppet master receives a catalog
request with a valid certificate, it begins a process called **node
classification** to determine what Puppet code will be compiled to generate
a catalog.

There are three different ways to handle node classification.

1. The `site.pp` manifest is a special file on the master where you can write
node definitions. This is the method we'll be using now and in several of the
following quests. It gives you the most direct view of how node classification
works.

2. The PE console includes a GUI node classifier that makes it easy to manage
node groups and classification without edit code directly. Though this is a
very effecient way to manage node classification, you'll understand it most
clearly after you're familiar with some of the underlying Puppet concepts.

3. Finally, if you want to customize node classification, you can create your
own [External Node
Classifier](https://docs.puppet.com/guides/external_nodes.html). An external
node classifier can be any executable that takes the name of a node as an
argument and returns a YAML file describing the Puppet code to be applied to
that node. This is an advanced topic, and won't be covered in this guide.

## The site.pp manifest

When a Puppet agent contacts the Puppet master, the master checks for any node
definitions in the `site.pp` manifest that match the agent node's name.

It will help to understand what a node block looks like with an example, so
go ahead and open your `site.pp` manifest.

    vim /etc/puppetlabs/code/environments/production/manifests/site.pp

Go ahead and scroll past the comments and default node definition to the bottom
of the file. This is where you'll create a new node definition for the
`learning.puppet.vm` node. The outline of the node definition should look like
this:

```puppet
node learning.puppet.vm {

}
```

Normally you would include the class or classes you want to include on this
node inside this node definition (hence the term "classification"). A class is
a block of Puppet code that defines a related group of resources. We'll cover
the details of classes in the next quest, but for now, we'll take a little
short cut and write a resource declaration directly into your `site.pp`
manifest. In this case, we'll use a resource type called `notify` that will
display a message in the output of the Puppet run without making any changes to
the system.

Go ahead and add the following `notify` resource to your node definition. (You
may notice that this resource declaration doesn't include any parameters. This
is because the only parameter we care about in this case is `message`, and
because `message` is the notify resource's namevar, it will default to the
resource title.)

```puppet
node hello.learning.puppetlabs.vm {
  notify { 'Hello Puppet!': }
}
```

(You'll probably learn Puppet code syntax more quickly if you type out your
code manually, but if you prefer to paste content into Vim, you can hit `ESC`
to enter command mode and type `:set paste` to disable the automatic
formatting.  Press `i` to return to insert mode before pasting your text.)

Remember, use `ESC` then `:wq` to save and exit.

<div class = "lvm-task-number"><p>Task 17:</p></div>

Now SSH to your agent node:

    ssh learningt@learning.puppet.vm

And trigger another puppet run

    sudo puppet agent -t

The output will include something like this:

    Notice: Hello Puppet!
    Notice: /Stage[main]/Main/Node[learning.puppet.vm]/Notify[Hello Puppet!]/message: defined 'message' as 'Hello Puppet!'
    Notice: Applied catalog in 0.45 seconds

Now go ahead and disconnect from your agent node.

    exit

## Review

We introduced the role of certificates in the master/agent relationship, and
the communication that takes place during a puppet agent run. Finally, you
defined a `notify` resource in the master's `site.pp` manifest and triggered
a puppet agent run on the agent node to enforce that configuration.