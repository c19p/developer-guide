# Architecture

![Architecture][Architecture]

As mentioned a few times on the [User Guide], the main components of a C19 agent are its three different layers, each is responsible for a specific task:
The `Agent` exposes ways for your application to set and get values to and from the state, the `Connection` layer is responsible for exchanging the state with 
other peers and the `State` is responsible for holding the state itself.

A user of the C19 protocol has the option to choose any of the different layers to work together. For example, they might choose an Agent that exposes HTTP endpoints 
for setting and getting values or one that exposes a Websocket connection. The State layer might be one that is backed up by a DB (e.g Redis) or one that holds the state 
in its own data structure in-memory.

In this chapter we will explore the different layers in a bit more details and talk about the common parts that hold them together.

[Architecture]: architecture.png "Architecture"
[User Guide]: [FIXME: link to user guide / architecture]
