# Query Paradigm

One of the benefits of the C19 protocol is the fact that it brings the data locally to the application. But if the data is too large to be exchanged in full then different 
solutions might come handy. One of the options is the Query Paradigm. 

We call it the `Query Paradigm` since it's a paradigm change from how the `Default` layers are implemented today. The `Query Paradigm` works like this:

When an application queries its local C19 agent, if the C19 agent doesn't have this data available it will send a query to a set of other C19 agents. Those 
agents will query other C19 agents if they don't hold the data themselves and will eventually return a results. The result can be cached to the local C19 agent 
and therefore create an on-demand data distribution where each C19 agent holds the data that is required by the application.

The rate in which a query propagates throughout the system can vary depending on a number of different parameters. For example, how many C19 agents are being queried 
on each cycle.

## The Layers
There are different ways to go about implementing this, but it seems like any solution would require introducing new APIs between the layers. For example, we 
would want the `Connection` layer to be aware when the `Agent` layer queries the `State` for a key and that key is not found. The `Agent` layer would have to 
"wait" for the query to be resolved by getting a signal from the `State` that the key was just committed. Consider the following scenario:
1. The application queries the local C19 agent
2. The `Agent` layer tries to get the value from the state
3. The values does not exist so the `Agent` layer subscribes for a notification for this key
4. The `Connection` layer gets a signal from the state that a key is missing and sends a query throughout the system
5. When it gets a response it will commit the key to the state (or either commit a "not found" signal)
6. The `State` layer will have to notify the `Agent` that the key was just committed
7. The `Agent` will return a response to the application

This can be done with an async API to the application or a sync one.

