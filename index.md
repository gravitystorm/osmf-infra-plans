---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

These are some ideas for the OSMF infrastructure that I've built up over the years. Other people will have different ideas, but these are mine.

# Background

I've been part of the OpenStreetMap Foundation's Operations Working Group (OWG) since before it was even called that, but in May 2019 I stepped down to concentrate on other OpenStreetMap projects.

When I first got involved, the OSMF infrastructure was a few machines in a rack in an old university researcher's office, and it has slowly grown over the years, moved out of the office and into datacentres in various countries. The infrastructure has grown organically, with no significant changes in strategy in the last decade. I believe that we've approached a limit in manageability, and that some changes in strategy will make it much easier to continue over the next few years.

At the time of writing (May 2019) the OSMF is managing 87 servers, and you can see the list of what they are an what they are for on the [OSMF Server Info](https://hardware.openstreetmap.org/) website. Which is, by the way, still my all-time most favourite example of what you can do with a bit of automation and static web pages. But that's a different story.

The servers can be split into roughly three groups:

* Tileserving. The largest group is the edge tile caches (35 machines), and another 7 machines are used for rendering.
* API and website. 3 database servers, 5 backends and 5 frontends.
* Other things. 32 machines doing everything else, some of which are critical for primary services, others of which are mostly idle and some completely unused.

For conflict of interest reasons, I'm not going to discuss the topic of tileserving in this document.

# One Thing Per Thing

We've evolved what I'll term a "One Thing Per Thing" approach to infrastructure, which is mostly running one instance of each service, and generally only having one or two services on each machine. This is a fairly traditional approach. For example, we have one machine running as a forum server - that's the only thing that machine does, and it's also the only machine involved in the forum. We have one machine running chef-server, and it's the only thing that machine does. There's a machine that only runs the GPS tiles, and GPS tiles only runs on that machine. You get the idea. Some services grow to the point that they outgrow their single machine. Then they can either be spread across multiple machines running as duplicates of each other (e.g. nominatim), or the machine itself is beefed up to cope (e.g. the 'core services' machine, or the main database servers). Sometimes, but rarely, we have multiple services on the same machine.

This approach has worked for years, let's not lose sight of that. But it also leads to a number of problems:

* Some of the machines are tremendously underused (or overpowered). We don't need 12 cores and 32 gigs of ram to run the forum, for example. But there's no way to shift any of the resource-intensive tasks between machines, and certainly no automated way to do so.
* Single points of failure are everywhere. If there's a technical problem with one machine, that service disappears until an admin can either fix its machine, or restore from backups to a spare machine elsewhere. If a machine needs to be relocated, then we need to plan for hours or days of downtime.
* No isolation of services. Some machines run multiple unrelated tasks, like running the mailing lists service on the same machine as the help system. But services can depend on different software versions, leading to upgrade problems, and mis-behaved services can interfere with other services, for example if a service goes awry.
* Any new thing is a big new thing. If a community member wants to try running a new service, they need an entire machine of their own for that particular service, and to forecast how big a machine that needs to be. That machine needs to be purchased, and hosted, and there's an ongoing commitment that needs to be carefully budgeted. So we tend to turn suggestion for new services away. This means that over the years, many key community services are run by different organisations instead of OSMF.

# Financing

One side note here is on the way we've budgeted and spent money over the years. A decade ago, we mostly had free hosting in universities, and often used donated hardware. OpenStreetMap had big needs, but OSMF had a small budget, and so any free or cheap options made sense. Moreover, the spending on hardware was very lumpy, and a single database server would account for a large fraction of the annual budget of the whole of the OSMF. So when we spent money on hardware, we ran donation drives, bought the hardware outright and put it in our free hosting.

In this way we traded large, occasional, [capex](https://en.wikipedia.org/wiki/Capital_expenditure) for smaller ongoing [opex](https://en.wikipedia.org/wiki/Operating_expense), which was entirely deliberate and I still think was a great approach. It matched the OSMF financing, which was mainly through unpredicatable donation drives, rather than their much smaller regular membership income, and so it reduced the risk to the whole project of large ongoing expenses causing cashflow problems. It's something that I fully supported and explained to many people over the years.

But times have changed. In particular, we've now go so much hardware that Capex is now routine and more predicatable. We now mostly buy our own machines, rather than using castoffs. We've outgrown our dontated hosting and our main datacentre is on fully commercial rates. Most OSMF income is now predicatable too, and individual hardware purchases are no longer breaking the bank. So we can rethink how we spend money, and what we spend the money on, and should not be as cautious about committing to opex as we used to be.

# Ideas from elsewhere

In the last decade there's been plenty of people working on similar issues all over the world. So we don't need to invent new ideas from scratch, we can take the ones that we like from elsewhere, and keep the ones that we still find useful. Again, doing things differently in the future doesn't mean that we were wrong in the past, just that circumstances have changed and strategy should change too.

# The goals

It's worth having a list of goals written down, before talking about the solutions, so here we go:

* Avoid service outages from individual hardware failures. This protects against emergencies, and also lets our tiny sysadmin group rest more easily at night.
* Reduce underused resources. There's no point in owning dozens of idle machines, since they use up OSMF money each month on hosting and power.
* Easier resource planning. We don't want a complex system with individually customised machines. No special machines with big disks and small memory for this one specific service. Standardised machines whereever possible.
* Cheaper. All the money OSMF spends on infrastructure could be spent elsewhere. So we shouldn't pick expensive options just for the sake of it.
* Easier to add new services. New ideas appear, new software gets developed, and we want to reduce the barriers to deploying the service and getting it used by the community. New services shouldn't require large costs.
* Easier to adjust. I would use the word 'scalable' but that often riles people up the wrong way. But we want to be able to add more disk space for one service, or more ram for another service, ideally at the click of a button or a few keystrokes in a config file.
* Maintain isolation of services. We put some services on different machines because they have a large attack surface, or in order to allow them to be administered separately.
* Reduce the sysadmin makework. We don't have many sysadmins, and some of the dull tasks can be avoided entirely, or traded for a few pounds a month to commercial providers. This allows the sysadmins to focus on the most important things that they do.

# Some ideas

This isn't meant to be comprehensive, and I'm sure there are other solutions too. But here are my own ideas on what [OWG](https://operations.osmfoundation.org/) should be prioritising.

## Storage

There's two main types of data storage that most services needs. These are databases, and somewhere to store other data like files. Currently every service that needs a database has their own, on the same machine, and stores all their files on the same machine too. Only the core database is different, with dedicated high-end database servers and a central file server, separate from the machines that run the service. I'll come back to that.

The industry has coalesced on using [object storage](https://en.wikipedia.org/wiki/Object_storage), for example Amazon S3, for most data needs. These work well over network connections, are easy to scale, and are fault tolerant to hardware failures. Around three years ago I suggested that we set up [Ceph](https://en.wikipedia.org/wiki/Ceph_(software)) on some spare hardware that we own, and use the S3-compatible API for object storage. We could then use this object storage for aerial imagery, for example, and for storing GPX files and user avatars from the main site, and it could be used as a safe place to store data for many of our other services. This will avoid the restore-from-backups scenarios for most of our services.

One key system to replace is the central NFS service. It's a key single point of failure for so many top-tier things the OSMF does, like the website, API and planet serving. All the data currently served to other machines via NFS should live on a replicated storage system like Ceph.

However, years have ticked past without progress on this topic, and not even a test system has yet been installed. There's now a profusion of S3-compatible options available from multiple suppliers, such as Digital Ocean and Exoscale, so we should consider whether these commercial services would work, if they would be cheaper, and if they would be easier to administrate. We could sign up for one of these options, start using it for some of our needs, and then move to our own in-house object storage if that makes sense in the future. But the key thing is to take one of the options and run with it.

## Smaller services

There's a long list of servers doing not very much, since the services that run on them aren't resource intensive. My long term goal would be to containerise them, but that's a lot of work and learning and so on, so shorter-term intermediate options should also be considered.

We could put these services in virtual machines, and run them on a smaller number of more powerful machines. That would make better use of resources, but introduces some complexity around running the hypervisor.

Or we could simply rent virtual machines from a variety of providers, like Amazon, Digital Ocean or Hetzner. This involves the least work, which is an important consideration for a small team. We can rent a virtual machine for a few pounds per month. These services don't really need to run on real hardware and they don't really need to run on OSMF owned hardware either. I would suggest doing this for chef, the forum, otrs, trac and similar services.

## Medium services

There's a few services where real hardware, rather than virtualisation, makes sense. Perhaps they need more disk space (and haven't yet been moved to use scalable storage, see above). But that doesn't mean that OSMF needs to buy the hardware, pay for the hosting, and deal with the associated hassle.

We could rent dedicated hardware from commercial providers instead. Even a few pounds extra per month over the lifetime of the server can be made up for reduced hassle for the sysadmins. For standard hardware, it would likely work out cheaper than owning, when hosting and warranties and so on are taken into account.

## Large services

These are things like the main database servers, the website and API service. These are fine as they are, for now at least. They still need to be managed, of course, but they can keep going as they are. It's often the first suggestion to move these 'to the cloud', but this is a great way to kill the conversation.

## Shared databases

We have seven different services that require MySQL, and five different services that require PostgreSQL. One of the latter is the main database, and that should stay separate. But the others should be consolidated onto central database servers, so that the services themselves can be moved around more easily. This would also allow new services to be set up without needing their own databases (and upgrades and backups thereof) as well as allowing services to scale up to multiple machines more easily.

We should also review the services that still use MySQL, and see how many can be migrated to Postgres. That would be one less thing to worry about.

## Containerisation

The world has moved to containers, for good reasons, and we should do so to. This would allow us to move services seamlessly from machine to machine, either for planned or unexpected outages. This would be an incremental project, and some services would need to wait until the storage and shared databases are available. But for me, this should be the eventual goal and a key target to aim for.

# The two or three next steps

A fully containerised set of services, with scalable storage options, running on a mixture of OSMF-owned hardware, rented hardware and rented virtual servers, would allow us to meet all of the goals that I listed above.

But the project can seem so large as to be fanciful, so let me detail what I believe are the next few steps:

* Set up, or rent, a storage service, and use it for two or three services.
* Nominate, and move, two or three services off of dedicated hardware and into virtual machines, either hosted on OSMF hardware or hosted commercially.
* Make it easier for external people to contribute. Our [chef cookbooks](https://github.com/openstreetmap/chef/) are already available, but not accessible. Configure two or three more cookbooks, including the wiki cookbook, to run locally on developers machines using test-kitchen.
* Get more people involved. If there's no volunteers available, then hire two or three freelancers as part-time assistants (two or three days per month) to help with sysadmins tasks.
* Measure progress. Unless two or three of these things are happening, find out why.
