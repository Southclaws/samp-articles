# SA:MP Cluster Servers on Kubernetes

Clustering, sharding or distributing is a systems engineering technique that has been used to provide highly available web services to large numbers of users.

There are a few large SA:MP server networks that have multiple numbered servers but I've never played these servers so I'm not sure if these servers are networked together and share state or if they are just a set of servers run by the same owners. I can imagine similar methods have been used in production in the SA:MP scene so I'd love to see some examples!

## What This will Acheive

Clustering a set of game servers across multiple logical servers using the techniques described here will offer the following benefits:

- Stateless server architecture
- Shared data and single point-of-truth state
- Failover recovery
- Sharding/Worlding features often seen in MMO games

## Why?

Why would you ever want to cluster your servers? Well, this only really applies to communities with *very large* player-bases up in the thousands. Once your userbase gets to this size, you really want to start thinking about your service availability (especially if you have business interests!)

If you're running a small-time server - anywhere from 0 to 100 players - this article probably won't serve any purpose aside from being interesting. So if you're still interested in clustered system architecture, read on!

### Statelessness

Stateless services are important to highly-available systems. MySQL has been common in SA:MP for many years and is on the right track (regardless of your opinions on databases!)

What "stateless" means in the context of a gameserver is slightly different to that of microservice architectures. For now, you can assume "state" refers to player account state not their internal game state (which cannot be directly accessed so we'll forget about it for the purpose of this post).

In a single-server world, if your server goes down, you want to be certain that when it comes back up, players can carry on where they left off. This has different meaning with different game types so for now we'll focus on Roleplay since that's where most of these large servers are. If you're running your MySQL database on the same logical server as your game server, you've already failed at being stateless!

In a cluster server world, the useful properties of stateless design become more apparent. If your shard goes down and players reconnect to a different shard, they should be able to continue doing what they were doing without even knowing they're on a different logical server.

### Single Point-of-Truth

Carrying on from the last paragraph, single point-of-truth means all your clusters are reading and writing to the same single logical database. I say "logical database" because you could also be clustering your data in order to be safe-by-redundancy, but that's a different article entirely!

Elaborating on the players reconnecting scenario, if your shards are all talking to the same database this means that they share the same state. This means you have a single account for a single player but they can log into that account on any shard in your cluster. This also means you must ensure a player cannot connect to two shards on the same account!

I assume I shouldn't really need to go on about the necessity of this. If all your servers have independent databases, you're not really clustering at all. It's important that, from a player's perspective, it's just one big server.

### Failover Recovery

I've mentioned this a bit already, servers crash. Hardware crashes. Drives get corrupted and generally, stuff just breaks!

If you're building a large system with lots of players, you really should be conscious of the possibility of stuff going wrong - hope for the best, plan for the worst!

So what happens if a cluster shard goes down?

- Your players should all implicitly join a new shard.
  - Yes this is possible without modifying SA:MP - you just need to be clever with your reverse proxy
  - All players should join the same shard - in a roleplay environment it's important that the group stays together.
  - You can either spin up a new shard or join them to an existing one with enough player slots - I'd opt for the former rather than the latter.
- Your players should be able to carry on where they left off.
  - Again, this relies entirely on a single point-of-truth design
  - How much state you want to maintain is up to you. You could store *every* important variable in a database.

I'll elaborate on these requirements later in the article, but this outlines most of what we will achieve with this design.

# Scratchpad (stuff not yet organised into the article)

single-front-multi-back will be done via a load-balancer, now how to go about this likely involves writing one from scratch (or at least writing a plugin for an existing load-balancer app) which will route player connections to backend based on instructions received.

so for example, the loadbalancer may receive an instruction to shift connections X, Y and Z from backend A to backend B - the game client will see a "lost connection to server" and "connecting to server" - the IP address is the same as far as the client is concerned, it's just the internal routing changes at the reverse-proxy stage where it routes TCP/UDP requests from the client to different backend servers which may be running on different logical servers in different parts of the world - this is how most modern SPA's like twitter or trello work.

in terms of seeing stuff from multiple servers, this isn't actually something I intend to do however you did give me an idea! I didn't think about different servers being for different areas of the map, so Los Santos could be Server A but as soon as the player crosses a city-limits boundary (a simple streamer plugin area) they could be reconnected to Server B which would be running the Red County area. The player would never see real players across this boundary but you could fake that with actors, and create some extra vehicles/objects from both servers near the boundary. However, another solution for that would be to borrow a great design from World of Warcraft which used a "Z" shaped hallway between instances so when the player is in the central part, they can't see anything on either server so a quick reconnect wouldn't make a difference.
