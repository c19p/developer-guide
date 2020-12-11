# Concepts

The C19 protocol's main goal is to bring the data local to the application. To reduce the need for an application to handle fetching data.
It does so by sharing state between different C19 agents.

The C19 agent is based on three layers: `State`, `Agent` and `Connection`. By using different combinations of different layer implementations, the user 
can run the protocol in many different ways that will solve for most of their use cases. One the of the major strength of the C19 protocol is being a 
single solution with different configurations to answer many use cases.

`The State` is where your data is being held. It allows the other layers to set and get values to and from the state.

`The Agent` is the entry-point to your application and where you communicate with the C19 agent. It exposes ways for your application 
to communicate with the C19 agent, set and get values to and from the state.

`The Connection` is the low-level layer that is responsible for communicating with other C19 agents and exchange the state with them.

The next chapter will deep dive into the architecture of the project and will show you how everything is connected.
