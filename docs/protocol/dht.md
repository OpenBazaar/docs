## Distributed Hash Tables

A distributed hash table (DHT) is a distributed key/value database. Unlike a traditional database which might store data on a single computer, a DHT distributes the data across nodes in a peer-to-peer network. Consider the following set of keys and values:

<img src="https://imgur.com/download/YzMdSyn">

If a user wants to store `Paul={"Computers, "Programming"}` in the DHT, he would connect to the peer-to-peer network and issue a `STORE` command to one or more nodes. Now anyone could retrieve that value by querying the network using the key `Paul` and they will get back `{"Computers, Programming"}`.

The trick here is making this DHT scalable so that it can handle large amounts of data. Having every node store all the data would scale very poorly. Therefore DHTs are designed to split the data set into shards and have only a subset of nodes store a shard. Once you do this the challenge then becomes figuring out which specific nodes to issue the `STORE` command to so that someone querying the network for a key can figure out which nodes they have to query to retrieve the data. 

There are different ways to accomplish this task and a number of different designs have been proposed and implemented. In OpenBazaar, the specific DHT design used is <a href="https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf">Kademlia</a>. 

#### Kademlia

Prior to joining a Kademlia DHT each node creates a unique identifier, or peer ID. Typically this can just be a long random number, but in OpenBazaar it's a SHA256 hash of an Ed25519 public key. This identifier also serves as an OpenBazaar user ID and corresponding public key allows the user to sign messages which can be validated against the ID.

The following is an example of a typical network topology. In this example, we've substituted the long SHA256 IDs for small integer IDs just to make it easier for us to visualize. Notice how the network is linear, with the smallest peer ID on the left and the largest on the right. In the actual OpenBazaar network, this range of possible IDs runs from 0 (on the left) to 2<sup>256</sup> (on the right).

<img src="https://imgur.com/download/CXWjd2o">

#### Routing Tables

Each node maintains a *routing* *table* which contains a list of peers (ID and IP address) that it knows about. We again run into scalability issues if each node attempts to store *all* nodes in its routing table. Therefore each node only stores a subset of the total nodes in its routing table giving it only a *partial* view of the network. 

However, it's not sufficient to choose which nodes to store or not at random. Instead, each node makes a decision whether to add a node to its routing table based on an algorithm. The goal of this algorithm is to ensure that the node has a fairly complete view of the other nodes that are "close" to it and a less complete, or partial, view of the nodes that "further" away.

What do we mean by "close"? We're not talking about physical distance (such as Node 14 is in Chicago and Node 38 is in Hong Kong). Instead, we are going to define *distance* in terms of each node's peer ID. One way to do this could be to use absolute value. For example, the "distance" between Node 14 and Node 38 would be 24 (|14 - 38| = 24). However, in Kademlia we use the xor(⊕) operation to determine distance. For example, the "distance" between Node 14 and Node 38, when measured by the xor metric, is 40 (14 ⊕ 38 = 40 and also 38 ⊕ 14 = 40).

Now that we have an idea what we mean by "close", we can proceed to define our algorithm. The routing table starts out containing only a single *bucket* which can hold at most *K* peers. The *K* parameter is selected as a tradeoff between efficiency and data redundancy. Typically it's set to 20 as research suggests that number is close to optimal. Every bucket has a *range* of peer IDs that it covers. For the sake of this example, let's suppose the range of our peer IDs is 0 to 99 (again a real network would be 0 to 2<sup>256</sup>).

Each time a node learns of a new peer (this tends to happen automatically since peers will contact each other periodically looking for data) it adds that peer to the bucket. If the bucket is full (meaning it already has 20 peers in it), the algorithm splits the bucket into two separate buckets as follows:

<img src="https://imgur.com/download/fnerbpB">

And the new peer would be added to the bucket whose range its ID falls within. For example, if we were adding peer 19, it would be added to bucket 0. Now that we have more than one bucket, the decision about what to do when a bucket gets full is slightly different. If the ID of the peer falls within a bucket whose range *our* node's ID also falls (we could call this our *neighborhood*), then we split the bucket just like in our previous example. If the bucket's range does not encompass *our* node ID, then we don't add the peer to our routing table. 

As an example, suppose our Node ID is 29. We fall within the range of bucket 0. Further, suppose we are considering whether we should add Node 74 to our routing table. Node 74 falls within bucket 1 but bucket 1 is full. In this case, we do *not* add Node 74 since it falls in a full bucket that is different from the bucket our Node ID would fall within. 

If we were to consider adding node 42 to bucket 0 but bucket 0 is full, we would *split* bucket 0 into two separate buckets because that is the bucket whose range our own ID falls within. 

Through this algorithm we ensure that only peers that are "close" to us are added to the routing table, while exponentially fewer peers are stored the farther away they are. Below we have an example of a routing table for a network with a range of 0 to 2<sup>160</sup>.

<img src="https://imgur.com/download/giOxx0u">

#### Bootstrapping

Let's go back to our original example. Suppose we've generated a peer ID 81; how do we actually join the network? Like all peer-to-peer networks, we need to know the IP address of at least a few nodes in the network in order to join. These IP addresses can be hard-coded in the software or could be fetched from a seed server. In either case, the first thing we're going to need to do is populate our routing table. How do we do this?

Suppose one of the bootstrap nodes we use is Node 5. What we're going to do is connect to Node 5 and say, "Give me the 3 nodes in your routing table that are closest to Node 81". And Node 5 would respond with the IP addresses of those three nodes, let's say Nodes 20, 26, and 38. This is called a `FIND NODE` command. Again, each time we learn of a new node, we add that node to our routing table according to the algorithm we described above.

Now we can go to Nodes 20, 26, and 38 and issue *them* `FIND NODE` commands and they will, presumably, respond with nodes that are closer still to ID 81. We can keep making these iterative `FIND NODE` queries (sometimes called a *crawl*) until the nodes we get back are no closer to 81 than the closest node in our routing table. At this point, we can stop the crawl and have successfully bootstrapped our routing table. 

<img src="https://imgur.com/download/BOgpLgM">

#### Storing Data

Suppose we want to store a key/value pair in the network. How do we do it? First, we hash the key with SHA256. Continuing from our example above, if the key we want to insert is "Paul" then we do:
```
key = SHA256("Paul") // 818b5cc5f21d3e6e4e6071c06294528d44595022218446d8b79304d2b766327a
```

Our goal is to find the *K* closest nodes (against K is usually 20) to the `key` and give them both the key and value to store by issuing a `STORE` command.

How do we find the 20 closest nodes to the key? By doing the same type of iterative `FIND NODE` crawl we did above. The only difference this time is we select the initial 3 nodes to query from our routing table (the 3 closest nodes to the key) instead of using the list of bootstrap peers. 

Upon completion of the crawl we should we should have the IP addresses of the 20 closest nodes and can issue `STORE(key, value)` or in our example, `STORE(818b5cc5f21d3e6e4e6071c06294528d44595022218446d8b79304d2b766327a, {"Computers, "Programming"})`.

#### Fetching Data
It should be fairly easy at this point to see how we will get the value back out of the DHT. If someone knows a key, "Paul" for example. Just like before, they can calculate:

```
key = SHA256("Paul") // 818b5cc5f21d3e6e4e6071c06294528d44595022218446d8b79304d2b766327a
```

However, instead of a `FIND NODE` crawl, they will do a `FIND VALUE` crawl. This type of command behaves just like a `FIND NODE` command except we give the remote peer the key we are looking for. If they have the corresponding value, they will return it to us. If not, they return the 3 closest peers just like in a `FIND NODE` command. By the end of the crawl, we should have the value if it existed in the DHT.

#### Ensuring Persistence

As we already mentioned, `STORE` commands are issued to *K* nodes instead of just one. This is to ensure that the data is replicated on more than one node and to guard against losing data when nodes go offline. 

In addition, nodes need to be programmed to proactively share values with new nodes as they join the network. When a node learns of a new node in its neighborhood, it should share any values whose keys are close enough to the new node that it should be storing them. 

In this way, a DHT can be said to be "self-healing" in that the network can withstand fairly high node churn and still keep values alive. 

#### The OpenBazaar DHT

While the value one stores in a DHT could be arbitrary data, such as images, product listings, or chat messages, in OpenBazaar we only store *pointers* to this data in the DHT. A pointer is not the data itself, but rather a list of IP address of nodes that have the value. In other words, a pointer *points* to the nodes that have the value. For example:
```
value = [
    {
        "peerID": QmNedYJ6WmLhacAL2ozxb4k33Gxd9wmKB7HyoxZCwXid1e,
        "addresses": [
            "/ip4/103.2.117.6/tcp/4001",
            "/ip4/127.0.0.1/tcp/4001",
            "/ip6/2001:0000:3238:DFE1:63:0000:0000:FEFB/tcp/4001",
            "/ip6/::1/tcp/4001"
        ]
    },
    {
        "peerID": QmamudHQGtztShX7Nc9HcczehdpGGWpFBWu2JvKWcpELxr,
        "addresses": [
            "/ip4/202.55.147.10/tcp/4001",
            "/ip4/127.0.0.1/tcp/4001",
            "/ip6/3ffe:1900:4545:3:200:f8ff:fe21:67cf/tcp/4001",
            "/ip6/::1/tcp/4001"
        ]
    },
    {
        "peerID": QmbyUYWZEBRFw9uxVThS4FYMwkdhWfGAsYwppBKTF6L968,
        "addresses": [
            "/ip4/192.231.203.130/tcp/4001",
            "/ip4/127.0.0.1/tcp/4001"
        ]
    }
]
```
There are several reasons why it is preferable to store pointers as values rather than the full data:

- Far more people can be storing the actual data. DHT data is only stored by *K* nodes that place a hard limit on the amount of data replication we can have. If we only store pointers in the DHT, there is no limit to the number of nodes who can store the data.

- Because each node has to regularly share its data with other nodes (such as when new nodes join the network), the DHT could end up using enormous amounts of bandwidth if it had to share larger files such as images or videos with such regularity. It's much less of a burden for a node to share tiny pointers.

- No node in the DHT is forced to store content which they do not wish to store. For example, if we allowed storage of arbitrary data, you could end up storing illicit or illegal content against your will, which could get *you* in trouble just for running the software.

Since OpenBazaar is only storing *pointers* in the DHT we generally refer to the DHT as our routing layer since it's used to route download requests to the appropriate nodes.

#### Seeding Files
In OpenBazaar one can "seed" a file by inserting the hash of the file into the DHT as the key and a pointer to one's node (peerID and IP addresses) as the value. The nodes that receive the `STORE` command will *append* the pointer to the list of pointers it is storing for that key. Anyone else can download the file if they know the hash by querying the DHT for the hash then using the returned pointers to connect to one or more of the peers seeding the file to download it. Seeders periodically re-publish their pointers to ensure persistence. 