# Description

Specification for federated social media protocol.

## Terminology

* **Instance**: a currently running server process, receiving
  HTTP requests/communicating with DB/etc.

* **Server**: logical unit of the fed system, consisting of 1
  or more instances and a single logical database.

* **Local server**: the server a user is registered with, and accessing with that server's frontend.

* **Remote server**: a server the user is not registered with,
  and interactions will be proxied through the local server
  using the protocol we are discussing.

* **Community**: collection of posts related by a common topic
  on a single server (a subreddit).

* **Post**: basic unit of content in our protocol, can be nested
  and frontends may choose to render top level posts as primary
  pieces of content, with nested posts rendered as comments.

* **User**: entity representing a student/member of staff registered with a server. Users cannot be transferred
  between servers, but can of course view and interact with
  posts and communities on remote servers.
