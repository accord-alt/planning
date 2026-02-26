# Accord

A plugin driven, p2p, discord like, alternative

## Tech stack

- Rust
- Flutter

## Architecture

- P2p nodes
- Library
- Clients

### P2p nodes

- Written in rust
- Uses libp2p
- Uses merkle trees for data storage and synching

#### Modes

- Gossip
- User

#### Gossip

##### Discovery

- Find peers
  - scan locally using mdns
  - scan remotely using kademlia
  - scans across port range: 51030-51042

##### Storage

- Stores peers
- Stores messages as a merkle tree
- Stores an index of the root hash of the merkle tree

```ascii
$HOME/.accord
├── peers (vector of peer addresses)
├── data (merkle tree storage)
└── index (merkle tree root hash)
```

##### Sync

- Syncs messages with peers

#### Full

##### Discovery

- Find peers
  - scan locally using mdns
  - scan remotely using kademlia
  - scans across port range: 51030-51042

##### Storage

```ascii
$HOME/.accord
├── peers (vector of peer addresses)
├── database (merkle tree storage)
│   ├── 1234567890 (merkle tree root hash)
│   │   └── data (merkle tree)
│   └── ...
├── current (current merkle tree root hash)
├── indexes (merkle tree of root hashes)
├── users (all known users)
│   ├── 1234567890 (user id)
│   │   └── meta (meta data as json)
│   └── ...
└── user
    ├── connections
    │   ├── 1234567890 (user id of connection)
    │   │   ├── meta (meta data as json)
    │   │   ├── private (generated private key from master private key)
    │   │   ├── public (generated public key from private key)
    │   │   └── shared (generate shared key from diffie-hellman key exchange)
    │   └── ...
    ├── id (generated id from eliptic curve)
    ├── meta (meta data as json)
    ├── private (generated private key as master key)
    └── public (generated public key from private key)
```

##### Sync

- Syncs messages with peers

##### Sync Workflows

```ascii
+================+==============================+=========+========================+
| Library        | Node                         | Storage | Network                |
+================+==============================+=========+========================+
| sync data ------->                            |         |                        |
|                | Read current ------------------>       |                        |
|                | Send current ---------------------------->                      |
|                |                              |         |                        |
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Already the latest data ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
|                |                              |         |                        |
|                |                       <----------------- Latest                 |
|              <--- Done                        |         |                        |
|                |                              |         |                        |
~~~~~~~~~~~~~~~~~~~~~~~~~~~ New data available on network ~~~~~~~~~~~~~~~~~~~~~~~~~~
|                |                              |         |                        |
|                |                       <----------------- Update with tree       |
|                | Get root hash of tree        |         |                        |
|                | Create directory with hash ---->       |                        |
|                | Store tree in directory ------->       |                        |
|                | Update current with hash ------>       |                        |
|                | Update indexes with hash ------>       |                        |
|              <-- Done                         |         |                        |
|                |                              |         |                        |
~~~~~~~~~~~~~~~~~~~~~~~~~~~ New data available in storage ~~~~~~~~~~~~~~~~~~~~~~~~~~
|                |                              |         |                        |
|                | Send update with tree ------------------->                      |
|              <-- Done                         |         |                        |
+================+==============================+=========+========================+
|                |                            <------------ Sync request with hash |
|                | Read current ------------------>       |                        |
|                | Check hash against current   |         |                        |
|                |                              |         |                        |
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Already the latest data ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
|                |                              |         |                        |
|                | Send back latest ------------------------>                      |
|                |                              |         |                        |
~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Does not match latest data ~~~~~~~~~~~~~~~~~~~~~~~~~~~~
|                |                              |         |                        |
|                | Read indexes ------------------>       |                        |
|                |                              |         |                        |
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ Hash exists in indexes ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
|                |                              |         |                        |
|                | Send update with tree ------------------->                      |
|                |                              |         |                        |
~~~~~~~~~~~~~~~~~~~~~~~~~ Hash does not exist in indexes ~~~~~~~~~~~~~~~~~~~~~~~~~~~
|                |                              |         |                        |
|                | Send current hash ----------------------->                      |
|                |                            <------------ Update with tree       |
|                | Get root hash of tree        |         |                        |
|                | Create directory with hash ---->       |                        |
|                | Store tree in directory ------->       |                        |
|                | Update current with hash ------>       |                        |
|                | Update indexes with hash ------>       |                        |
+================+==============================+=========+========================+
```

##### Message Workflows

```ascii
+==================================+=================+=========+======+
| Library                          | Node            | Storage | Sync |
+==================================+=================+=========+======+
| create message <message model> ---->               |         |      |
|                                  | Store message ---->       |      |
|                                  | Sync data ------------------>    |
+==================================+=================+=========+======+
```

##### User Workflows

```ascii
+=================+========================================+=========+======+
| Library         | Node                                   | Storage | Sync |
+=================+========================================+=========+======+
| create user ------>                                      |         |      |
|                 | Create ID                              |         |      |
|                 | Store ID -------------------------------->       |      |
|                 | Create master private/public key pair  |         |      |
|                 | Store master private/public key pair ---->       |      |
|                 | Sync user id with peers --------------------------->    |
|               <-- Return user as user model              |         |      |
+=================+========================================+=========+======+
| get users -------->                                      |         |      |
|                 | Get users ------------------------------->       |      |
|                 | Build user models from id and meta     |         |      |
|               <-- Return list of user models             |         |      |
+=================+========================================+=========+======+
| get user <id> ---->                                      |         |      |
|                 | Get user -------------------------------->       |      |
|                 | Build user model from id and meta      |         |      |
|               <-- Return user model                      |         |      |
+=================+========================================+=========+======+
```

##### Connection Workflows

```ascii
+===============================+========================================+=========+=========+====================================+===================+
| Library                       | Node                                   | Storage | Sync    | User node                          | User Node Storage |
+===============================+========================================+=========+=========+====================================+===================+
| create connection <user id> ---->                                      |         |         |                                    |                   |
|                               | Create Connection directory with id ----->       |         |                                    |                   |
|                               | Create a private/public key pair -------->       |         |                                    |                   |
|                               | Share public key via message ---------------------->       |                                    |                   |
|                               |                                        |         | Syncs ---->                                  |                   |
|                               |                                        |         |         | Accepts connection                 |                   |
|                               |                                        |         |         | Create a private/public key pair ---->                 |
|                               |                                        |         |         | Create a shared key ----------------->                 |
|                               |                                        |         |       <-- Send public key via message        |                   |
|                               |                                      <------------ Syncs   |                                    |                   |
|                               | Create a shared key --------------------->       |         |                                    |                   |
+===============================+========================================+=========+=========+====================================+===================+
```

### Library

- Written in rust
- Uses public/private keys for message encryption and user authentication
- Has a plugin system

#### Models

##### Message

- Has
  - An id using UUID
  - A from ID
  - A to ID
  - A plugin type
  - A plugin body
  - A created at timestamp with time zone
  - An updated at timestamp with time zone
  - A deleted at timestamp with time zone

##### User

- Has
  - A master private/public key pair
  - An ID using ecliptic curve
  - Meta data using json
  - Many messages
  - Many connections

##### Connection

- Has
  - A private/public key pair
  - Meta data
  - A from ID
  - A to ID

#### Plugins

- Read incoming messages
- Find plugin by plugin type from message
- Execute plugin with the message as an argument
