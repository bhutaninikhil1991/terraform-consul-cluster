# Creating a Resiliant Consul Cluster for Microservice Discovery with Terraform and AWS

[Consul](TODO) is a technology will enables *Service Discovery*, a pattern which allows services which are created on arbitrary hosts and expose APIs on arbitrary ports to communicate with each other, via a central authority.

This kind of pattern is critial in the world of *microservices*, where many small services will be running on a cluster. Services may die, due to errors or failing hosts, and be recreated on new hosts. In these cases, it is essential that the system as a whole has a registry of where each service lives and how to address it. It is also crucial that this registry is *resilient*, because if it is unavailable, then it becomes extremely challenging for the system to operate successfully.

Creating such a resilient system can be complicated, ideally when doing so we want to build the system in a *reproducible* manner, and also be able to change the system in a very controlled fashion. This is where *Terraform* comes in. Terraform is a technology which allows us to *script* the provisioning of infrastructure and systems, then execute the provisioning using simple, automatable processes. This means that changes to complex systems such as the one we build can be expressed in a form that have been invaluable in traditional software development for years - code diff, or commonly, pull requests. We get version controlled, tracked changes our system, with documented details on what changed and why.

And why [AWS](TODO)? We need to create many servers and build a network to see this system in action. We can simulate parts of this locally with tools such as [Vagrant](TODO), but we can use the [most popular](Citation Needed) IaaS platfom for this job at essentially zero cost, and learn some valuable skills which are readily applicable to competing projects at the same time. The techniques involved would not be dissimilar on a platform like Azure, and by using Terraform, which is not bound to a particular provider only, we can apply many of the same principles.

## The Goal

The goal is to create a system like this:

TODO - system diagram

In a nutshell:

- We have a set of homogenous Consul nodes
- The nodes form a cluster and automatically elect a leader
- The nodes are essentially homogenous, we do not need to create a leader node or follower node, nodes handle this at runtime
- The nodes span more than one availability zone, meaning the system is redudant and can survive the failure of an entire availability zone (i.e. data centre)
- The Consul administrive interface is available to view via a gateway
- We have two example microservices which register themselves on the cluster, so we can actually see some registered services in the console

As a quick caveat, in reality this setup would typically live in a private subnet, not directly accessible to the outside work except via public facing load balancers. This adds a bit more complexity to the Terraform setup but not much value to the walkthough. A network diagram of how it might look is below, I invite interested readers to try and move to this model as a great exercise to cement the concepts!

## Step 1 - Creating our Network

The first logical step is to create the network itself. This means:

- The VPC (Virtual Private Cloud)
- The 'public' subnet, which defines our IP ranges for hosts
- The internet gateway, which provides an entry/exit point for traffic on the internet
- The firewall rules, which define what traffic can come in and out of the network

All together, that's this:

TODO - network diagram

Creating a VPC and building a subnet is fairly trivial if you have done some network setup before or spent much time working with AWS, if not, you may be a little lost already. There's a good [course on Udemy which will take you through the process of setting up a VPC](TODO) which I encourage you to look into if you are interested in this. It'll also show you how to build a more 'realistic' network, which also contains a private subnet and NAT, but that's beyond the scope of this write-up. Instead, I'll take you through the big parts.

### The VPC

A VPC is a Virtual Private Cloud. Think of it as a whole network which you will build to play with. The key thing is that it is *isolated*. Things you create in this network will be able to talk to each other if you let them, but cannot communicate with the outside world, unless you specifically create the parts needed for them to do so.

You probably use such a network regularly if you work in a company. Most companies have their own internal network - when you use a computer on that network it can talk to other company computers (such as the company mail server). When you are off that network, you might not be able to access your company email (unless it is publically availale, like gmail, or over a VPN [and by accessing a VPN, you are actually *joining* the network again, albeit remotely]).

Perhaps the most immediately obvious part of a VPC is that *you control the IP addresses*. You specify the *range* of IP addresses which are available to give to machines on the network. When a machine joins, it must have an IP address in that range. It can be assigned dynamically or statically. I'm not going to go into too much detail here, if you are interested let me know and I'll write up an article on VPCs in detail:

TODO digram with some hostsc

How do we script the creation of a VPC? If you are new to this, my suggestion is:

1. Use the AWS console to create what you want
2. Search the Terraform documentation for the entity you want to create (e.g. [VPC](TODO)), script the creation and apply the provisioning
3. Compare the hand-made VPC to the script-made VPC, if the two are alike, you are done
4. If the two are different, check the documentation and try again

Ensure you have an AWS account, and note your Secret Key and Access Key. We'll need these to remotely control it. Here's the terraform script to create a VPC:

```
TODO
```

This script uses [Terraform Variables](TODO) which we keep in a [variables.tf](./infrastructure/variables.tf) file. Terraform will use the default values defined in the file if they are present, or ask the user to supply them. Let's build the network:

```
terraform apply
```

After supplying the values for the variables, Terraform will provision the network, using the AWS SDK internally.

### The Public Subnet

You don't put hosts directly into a VPC, they need to go into a structure called a 'subnet', which is a *part* of a VPC. Subnets get their own range of the VPC's set of available IP addresses, which you specify. They are used to build *zones* in a network. Why would you need this? Typically it is to manage security. You might have a 'public zone' in which all hosts can be accessed from the internet, and a 'private' zone which is inaccessible directly (and therefore a better location for hosts with sensitive data). You might have an 'operator' zone, which only sysadmins can access, but they can use to get diagnostic information.

The defining characteristics of zones is that they are used to create *boundaries* to isolate hosts. These boundaries are normally secured by firewalls, traversed via gateways or NATs etc.

TODO digram of subnet

This is build with the following snippet:

TODO

Note that we need a subnet for each availability zone. Why? They're in different datacenters, as we continue we want control over where we place hosts.

### The Internet Gateway, Route Tables and Security Groups

This article is not about creating a VPC, so I'm going to let you walk though the [./infrastructure/network.tf](TODO) script yourself at this point to see what's going on. We now have the following network:

DIAGRAM

TODO document final parts.

## Step 2 - Creating the Consul Hosts

Consul is designed to work in a cluster, to support high-availability. Three is a good number, five is probably overkill for many scenarios, but that's what we'll go for. Now as we want resilience, we want to make sure that if a host dies (for example, due to failure of a network controller or hard disk) we automatically create a new one. Enter [Auto-Scaling Groups](TOD). Auto-scaling groups allow us to define a template for an instance, and ask AWS to make sure there are always a certain number of the instances. We can go further and make this dynamic, making the number higher if we have greater load, or lower during quiet hours, but for now we can simple say 'Ensure we always have 5 instances':

```
TODO
```

So things are a bit more complex now. We have:

- An auto-scaling group, which demands we have a minimum and maximum of five instances
- A launch configuration, which tells the auto-scaling group what a newly created host should look like
- A 'setup' script, which runs on each newly created instance, and starts the consul service
- A load balancer, which monitors whether the admin UI is available on each instance

The load balancer include a heath check. If the host doesn't return an HTTP 200 for the admin UI 5 times in a row, with a gap of 30 seconds between each check, it will be marked as unhealthy and then destroyed. The auto scaling group will see we have too few hosts and create a new one, following the launch configuration.

As an added bonus, the load balancer takes in request on HTTP 80 and passes them to the admin UI port and address. If we run `terraform apply`, we'll see the IP address of the load balancer:

TODO

If we open it, we see our admin UI:

But we are only seeing one host. If we refresh again and again, we'll see different hosts each time, but they're not aware of each other:

TODO screenshot

This is because the hosts don't know about each other. When starting the Consul service, we need to provide a 'leader' IP - this is the instance each node in the cluster will talk to to establish a Consul cluster. This is the next part of the challenge.

## Step 3 - Determining a Leader

There's a nice trick we can use here. We can ask AWS to give us the IP addresses of each host in the auto-scaling group. We then pick an arbitrary IP address to be the leader. Each node needs to pick the *same* leader. A trivial way to do this is simply pick the lowest IP address of the set:

TODO diagram of election

Once this happens, the Consul nodes all know about each other. If the leader dies, the remaining nodes will elect a new leader. If we add a new node, it'll assume the leader is the lowest IP address. This won't always be the case, but that doesn't matter, because the node it talks to will know who the *real* leader is and handle the joining process for the new node.

**IMPORTANT NOTE**

This sounds simple and workabble. However, cluster management and leader election is an *exceptionally* tricky thing to get right and handle every edge case. See [TODO](). I believe this approach is robust, but cannot guarantee it covers every use case. YMMV.

The key changes are in the bash script to set up a clean node:

```bash
TODO
```

However, given that we use some AWS services here, we actually need to update our security policies, to allow the nodes to query the information they need to, such as the auto-scaling group instances. This means our [consul.tf](TODO) file gets a little more complex.

Let's destroy the stack and build it up again:

```
terraform apply
```

Now when we hit the load balancer, we should see a cluster of five nodes:

TODO

## Step 4 - Adding some sample microservices

I've created a docker image for as simple a microservice as you can get. It returns a quote from Futuram's Zapp Brannigan:

TODO

We can run two or three separate instnaces of this in the public subnet to simulate having some 'real' microservices. We use the [registrator]() tool to tell them to register with our Consul cluster.

The terraform script is quite simple and lives at `./infrastructure/sample-microservices.tf`.

Now when we run `terraform apply` and navigate to our Consul interface, we can see our services.

TODO

Looking good - it seems we have a working cluster. But how resilient is it?

## Step 5 - Spanner Throwing

Let's run some tests to see how resilient we are. We should at the very least try:

1. Killing a consul node, does it build a new quorum automatically and correctly? How long does it take?
2. Killing the consul leader, same question as above?
3. Killing 4 nodes at once, can we get back to a quorum?
4. Spend one hour killing a random node every five minutes, do we maintain a quorum whenever possible?
5. Kill every node, see what happens!

[Netflix](TODO) popularised tests like this with programs such as [Chaos Monkey](TODO) which run around a network killing things, but this is not a new practice. When designing systems it's great to play these kind of games; try and break the system, find novel ways to exercise its robustness. We don't necessarily need to be able to survive each scenario, if there are scenarios we know we cannot recover from automatically, that's fine, at least we can build a playbook for the devops team so we can be prepared.

Here's a snippet which kills a random node:

```bash
```

I've used bash to do the work, more sophisticated tests might be better written in code. The rest of the tests are available to look over in the [./tests](TODO) folder.

## Wrapping Up

There's a lot to see here, particularly if you are not particularly familiar with AWS. However, this should provide a good starting point to think about building your own resilient and robust systems. Interesting areas to look into to extend the project might be:

1. Automating the tests: Programatically monitor the cluster membership, if each test doesn't eventually result in a cluster of five nodes, the test fails.
2. Scaling by load: Reduce the quorum to three when load is low, scale up to five only as needed.
3. Breaking out the script: Instead of using a `user data` script to set up a node, bake it into a new custom AMI with [Packer]()

If you find this article, please do let me know. Any questions or comments are welcome!