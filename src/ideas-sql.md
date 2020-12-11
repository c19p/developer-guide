# SQL Backed State

The [Redis] use-case shows how we can have `Redis` as the backend of the `State`. While `Redis` is an excellent choice, it doesn't have to be `Redis` 
and can actually be anything that can hold data.

Imagine `SQLite` as the backend of the `State`. This means that the data can be stored in a relational database and the payload of the data can 
be a JSON object that represents a rows in the database or a set of rows. Of course if doesn't have to be JSON and can be any format you see fit.

As with the `Redis` use-case, the `Agent` can be agnostic to the fact that the backend is an SQL database so the `Default` agent can be used.
