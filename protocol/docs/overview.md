## High level Overview
OpenBazaar is the product of merging several advanced peer-to-peer or distributed technologies. Specifically, it uses a modified IPFS node (which combines ideas from Git, BitTorrent, and Kademlia), the digital currency Bitcoin, and Ricardian Contracts.

#### Seeding Content
When a user opens an OpenBazaar store, they *seed* their store data (the product listings, images, etc) in a way that might be familiar to anyone who has ever used BitTorrent. Potential buyers can view this store by making a direct connection to the seller and downloading the available product listings. Like BitTorrent, once a user downloads content from other peers, they start seeding that content as well. Through this process, user data quickly becomes replicated on multiple nodes in the network. The next time someone attempts to view that content, he can download it from any of the users who have it, not limited to only the originating node. For larger files users can download pieces from multiple peers simultaneously, increasing the speed of the download. 

This network architecture causes downloads to behave differently than the traditional client-server model. Whereas downloads in a client-server model slow down when the server comes under heavy load, downloads in OpenBazaar actually speed up in such circumstances. This is due to the sharding of files into smaller pieces and nodes sharing them as soon as they are downloaded, without waiting for the full file to download. 

Replicating data across multiple nodes also makes the data more resistant to censorship than the traditional client-server model. Taking a store offline isn't simply a matter of locating a server and performing a denial of service attack. In our case, the data can persist whether the originating node is online or not. 

#### Contracts
If a buyer decides to make a purchase, a signed order message is sent to the vendor (the vendor can be offline at this point as there is a means to recover messages sent to you while parties are offline). The buyer has the option of paying for the order by sending bitcoins either directly to the vendor or by funding an escrow address (see Escrow below). The orders are recorded in a cryptographic data structure known as a Ricardian Contract. Any further messages sent between the buyer and vendor, such as the order confirmation or order fulfillment messages, are recorded in the contract providing a cryptographic record of the order's history. 

#### Escrow
Unlike centralized eCommerce platforms, in OpenBazaar there isn't any company that can provide arbitration services if a contract is breached. However, this ends up being a strength of the platform as it allows for the creation of a market for such arbitration services. In OpenBazaar, the term "moderator" is used to represent a user who offers escrow/arbitration services to users of the network. 

Vendors have the option to select one or more moderators when creating listings. The vendor's choice of moderator(s) should be considered part of the bundle of goods offered up for sale. If a buyer does not recognize any of the moderators or does not believe any of them to be trustworthy, then he should not purchase the item. 

When a buyer makes a moderated purchase, the funds go into a bitcoin escrow address. Unlike traditional escrow, where a single person (or group) is trusted to keep the funds safe (and not steal them!), funds locked in a bitcoin escrow address cannot be removed from escrow without the cryptographic signatures of two of the three parties involved (buyer, vendor, and moderator). This requirement prevents theft in the event an untrustworthy moderator is chosen. 

If the order is completed successfully the buyer and vendor can sign to release the funds from the escrow to the vendor. If either party is unsatisfied, they can open a dispute with the moderator. The moderator, in combination with either the buyer or vendor, can sign to release the funds to whichever party wins the dispute. 
  
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

## IPFS
<a href="https://ipfs.io/">IPFS</a> stands for InterPlanetary File System. It is a hypermedia distribution protocol which forms the core the OpenBazaar network. It uses a Kademlia DHT, exactly as described above, to route downloaders to those seeding files. What makes it unique is how IPFS serializes the data to create a cryptographically authenticated data structure known as a *Merkle* *DAG*. 

[Note: much of this description of IPFS is taken verbatim from <a href="http://whatdoesthequantsay.com/2015/09/13/ipfs-introduction-by-example">Christian Lundkvist</a> since he did such a great job]

#### IPFS Objects

Before data is seeded it is wrapped in an IPFS object. Objects have two fields:

- `Data` - a blob of unstructured binary data of size < 256 kB.
- `Links` - an array of Link structures. These are links to other IPFS objects.

A Link structure has three data fields:

- `Name` - the name of the Link.
- `Hash` - the hash of the linked IPFS object.
- `Size` - the cumulative size of the linked IPFS object, including following its links.

IPFS objects are referred to by their hash, which is encoded in a Base58 multihash format. For example, `QmarHSr9aSNaPSR6G9KFPbuLV9aEqJfTk1y9B8pdwqK4Rq`. 

So an IPFS object may look something like this:
```
{
  "Links": [
    {
      "Name": "AnotherName",
      "Hash": "QmVtYjNij3KeyGmcgg7yVXWskLaBtov3UYL9pgcGK3MCWu",
      "Size": 18
    },
    {
      "Name": "SomeName",
      "Hash": "QmbUSy8HCn8J4TMDRRdxCbK2uCCtkQyZtY6XYv3y7kLgDC",
      "Size": 58
    }
  ],
  "Data": "Hello World!"
} 
```
It should first be noted that since IPFS objects are referred to by their hash, this data structure is cryptographically authenticated. If I fetch an IPFS object from the DHT by using its hash, I can verify that the data the peers returned to me has not been tampered with. The same goes for each the "links" inside the object. Once I download the parent IPFS object, I can proceed to fetch each of the links from the DHT by using their hash and validate them as well. Technically, since each link also contains a `Name`, our software can actual be told to fetch linked objects by their `Name` since it can always look up the corresponding `Hash` in the parent object. 

When we can download a file from anyone on the network by only knowing its hash, we call this *content* *addressing*. This is different from traditional HTTP requests that use *location* *addressing* ― fetching content from a specific location (such as a server).

Let's create a visualization of the above IPFS object:

<img src="https://imgur.com/download/56T4pfc">
 
#### Small Files
Small files (<256 kB) are represented as an IPFS object with the file data in the `Data` field and no `Links`. For example, a text file that says "Hello World" would look like this:
```
{
  "Links": [],
  "Data": "\u0008\u0002\u0012\rHello World!\n\u0018\r"
}
```
And in a more visual form:
<img src="https://imgur.com/download/m2VwTzR">

#### Large Files
Files >256 kB in size are split into chunks no larger than 256 kB and these chunks are linked to by the parent IPFS object (with filenames omitted). For example:

```
{
  "Links": [
    {
      "Name": "",
      "Hash": "QmYSK2JyM3RyDyB52caZCTKFR3HKniEcMnNJYdk8DQ6KKB",
      "Size": 262158
    },
    {
      "Name": "",
      "Hash": "QmQeUqdjFmaxuJewStqCLUoKrR9khqb4Edw9TfRQQdfWz3",
      "Size": 262158
    },
    {
      "Name": "",
      "Hash": "Qma98bk1hjiRZDTmYmfiUXDj8hXXt7uGA5roU5mfUb3sVG",
      "Size": 178947
    }
  ],
  "Data": "\u0008\u0002\u0018��* ��\u0010 ��\u0010 ��\n"
}
```
<img src="https://imgur.com/download/jBkCMdB">

When downloading large files, we don't have to download all links from the same node. Instead, we can download the links concurrently, from separate nodes, and dramatically increase the download speed. 

#### Directories

It's not hard to see how IPFS objects could be used to represent a file directory. Consider the following directory structure:

```
.
|--test_dir:
|  |--my_dir:
|  |  |--my_file.txt
|  |  `--testing.txt
|  |--bigfile.js
|  `--hello.txt
```
The files hello.txt and my_file.txt both contain the string Hello World!\n. The file testing.txt contains the string Testing 123\n.

When representing this directory structure as an IPFS object it looks like this:
<img src="https://imgur.com/download/0PM5xk9">

#### Versioning
IPFS can represent the data structures used by Git to allow for versioned file systems. A `Commit` object has one or more links with names parent0, parent1 etc pointing to previous commits, and one link with name object (this is called tree in Git) that points to the file system structure referenced by that commit.

We give as an example our previous file system directory structure, along with two commits: The first commit is the original structure, and in the second commit we’ve updated the file my_file.txt to say Another World! instead of the original Hello World!.

<img src="https://imgur.com/download/9Nbhvaw">

#### IPFS in OpenBazaar
In OpenBazaar we store all user data ― profiles, listings, product images, reviews, channels, etc ― in a directory referred to as the user's `root` directory. All of the user's files are stored either in the `root` directory or any of its subdirectories. This `root` directory is seeded on the network along with all the directory's files and subdirectories. By seeding data in this manner, we only need to know the hash of a user's `root` directory in order to download and view all the content that makes up the user's page or store. 

For example, an API call of `ipfs/QmfHTiFpqLDAVj29Nf7LrfUFfz4envqArY4Gv7CvbyDcPt` allows us to look inside a root directory:

<img src="https://imgur.com/download/xicvjy1">

And if we wanted to look inside the `listings` subdirectory we could call: `ipfs/QmfHTiFpqLDAVj29Nf7LrfUFfz4envqArY4Gv7CvbyDcPt/listings/`

<img src="https://imgur.com/download/VXmDvxe">

Note that in the above API call, we only needed to use the `Name` (/listing) and not the hash of the listing directory since the software will look up the `Hash` from the `Name`.

And finally we could fetch the data for the `cool-t-shrit` listing with the following call: `ipfs/QmfHTiFpqLDAVj29Nf7LrfUFfz4envqArY4Gv7CvbyDcPt/listings/cool-t-shirt.json`

## IPNS

Thus far we've seen how we can fetch a user's content given the hash of his `root` directory, but we have a bit of a problem. The `Hash` of the `root` directory changes every time we change the existing files or add new data. If we gave out our `root` hash to people so they can view our store, the hash would be made obsolete the next time we updated any data in our root directory (such as changing the price of a listing). 

IPNS stands for Interplanetary Naming System. It is a self-authenticating namespace built on top of IPFS. What we can do with IPNS is cryptographically map the hash of our `root` directory to our `peerID`. This is accomplished by signing the hash of the `root` directory with our identity key (remember the Ed25519 key we mentioned earlier) and insert this signed hash into the DHT using our peerID as the key. 

So inside the DHT we have a record that looks like:

```
peerID = signed(rootHash)
```

And since our peerId is the SHA256 hash of our RSA public key, anyone can fetch the latest copy of our `root` hash from the DHT and validate the signature against our public key, which itself should hash to our `peerID`. 

In this manner, one only needs to know our `peerID` to download an authenticated copy of all of our store content. 

Using the IPNS protocol the above API call which fetched the listing could be rewritten as `/ipns/QmdHkAQeKJobghWES9exVUaqXCeMw8katQitnXDKWuKi1F/listings/coot-t-shirt.json` where `QmfHTiFpqLDAVj29Nf7LrfUFfz4envqArY4Gv7CvbyDcPt` is our `peerID`.

And by using other naming protocols such as <a href="https://blockstack.org/">Blockstack</a> we can cryptographically map a user's `peerID`, which is a rather ugly looking series of numbers and letters, to a more human readable username such as `@UrbanArt`. Thus one only needs to know the human readable username to download user content in a cryptographically secure manner. 

## Bitcoin
The digital currency Bitcoin is used in OpenBazaar as the primary means of payment. The reason for this choice is two-fold. First, it aligns well with the goals of the project to be a decentralized, censorship-resistant eCommerce platform without a middle man. Bitcoin is also decentralized, censorship-resistant, and has no middlemen. It also has low transaction fees compared to other forms of payment. Presently a bitcoin transaction can be made for about 10¢ USD compared to about 30¢ *plus* 2.9% of the total for PayPal. Credit cards similarly take a percentage of the transaction. Due to their centralized nature, those methods of payment can also be easily tracked and censored.

The second reason Bitcoin is a good choice is it allows us to build the trustless escrow system we mentioned earlier. All other forms of payment require you to use the payment provider (such as PayPal) for arbitration if you have a dispute. With Bitcoin, we can not only create a free market for escrow/arbitration services, but we can do so in such a way as to remove the risk that the escrow agent will steal (or lose) the funds. 

#### Multisig scripts
If you like to learn more about how Bitcoin works we'd suggest reading the <a href="https://bitcoin.org/en/developer-guide">Bitcoin Developer Guide</a>. However, we can provide a quick overview of how the escrow system works. 

In Bitcoin, the coins are not technically sent to a bitcoin "address" or account. Instead, they are sent to a simple computer program (or script). This script sets the terms upon which the coins are allowed to be transferred. A person seeking to spend bitcoins provides the inputs to the script function and the bitcoin software will execute it. If the script returns `True` (and all other transaction checks pass) then the bitcoins may be transferred to another script. 

The specific script we use looks something like this:
```
OP_HASH160 <Hash160(redeemScript)> OP_EQUAL
```

Technically this script means "anyone who knows a certain password can spend these coins." Bitcoin underwent a soft-fork upgrade several years ago which gives this script a "special" meaning. In essence, when the interpreter sees this script it interprets it not as a password script, but as something called "pay to script hash" or P2SH.

Coins sent to this script can be spent by providing a `redeem script` whose hash matches the hash in the output script and then by fulfilling the terms of the `redeem script`.

In OpenBazaar we use a redeem script that looks like:

```
OP_2 <buyer_pubkey> <vendor_pubkey> <moderator_pubkey> OP_3 OP_CHECKMULTISIG
```

This script says the funds may be transferred if signatures matching two of the three listed public keys are provided.

The scripting language is flexible enough that we could extend it with additional features in the future. For example, suppose we want to add a timeout to the escrow. That is, if the buyer doesn't release the fund or file a dispute within 60 days, the funds will then be transferred to the vendor. Essentially this can save the vendor some headaches trying to collect his payment. 

This redeem script would look like:

```
OP_IF
    OP_2 <buyer_pubkey> <vendor_pubkey> <moderator_pubkey> OP_3 OP_CHECKMULTISIG
OP_ELSE
   "60d" OP_CHECKSEQUENCEVERIFY OP_DROP
    <vendor_pubkey> OP_CHECKSIG
OP_ENDIF
```

#### Bitcoin Wallets
The OpenBazaar protocol specification has nothing to say about which Bitcoin wallet should be used with the protocol. To improve the user experience the reference implementation comes bundled with a built-in wallet. The default wallet implements something call Simplified Payment Verification (SPV) which provides strong cryptographic validation of incoming Bitcoin transactions while using very little of the computer's resources. The drawback to SPV mode is it leaks enough private data to allow potential attackers to figure out which transactions came from the wallet. That information by itself doesn't say who the *owner* of the wallet is, though other investigative techniques might provide that information. 

For this reason, there is a setting in the openbazaar-go config file that allows a user to use bitcoind (a full Bitcoin implementation) with openbazaar-go. Bitcoind is a very heavyweight software and is typically only used by power users, but it does a much better job than SPV at providing transactional privacy. 

#### Altcoins
There isn't anything Bitcoin-specific about Kademlia, IPFS, or Ricardian Contracts. In theory, the OpenBazaar protocol could work with any digital currency, not just Bitcoin. In practice, there are certain features a digital currency must have to be used with the OpenBazaar protocol. For example, an altcoin must support multisignature transactions or the escrow system will not work. The protocol also makes some assumptions about the existence of payment addresses and transaction inputs and outputs. These assumptions could probably be abstracted away in future versions of the protocol, but it stands to reason that altcoins that are a close derivative to Bitcoin would work better with the OpenBazaar protocol than coins that are a drastic departure from it. 

## Ricardian Contracts
A traditional contract is a written or spoken agreement among two (or more) parties to exchange something of value. Every time we transact for anything we are entering into a legally binding contract, even if they are only verbal. When you purchase things on the internet, you are likewise entering into a legally binding contract. However, contracts are sometimes poorly written, ambiguous or difficult to interpret. They may be subject to *frog-boiling* where a strong party attempts to change the contract over time in his favor. Parties may even deny they agreed on a contract at all. These issues can make it difficult for arbitrators to determine who is correct in a dispute.

A Ricardian Contract is a type of cryptographic contract that attempts to solve these problems. Ricardian contracts are both human readable and machine parsable and provide an irrefutable record of what both parties agree to. It's not clear whether a Ricardian Contract would be treated as a valid contract in court (it would likely vary by jurisdiction) but it doesn't matter as the terms of the contract can be enforced programmatically by software. 

In OpenBazaar the Ricardian Contract looks as follows:
```protobuf
syntax = "proto3";

message RicardianContract {
    repeated Listing vendorListings                    = 1;
    Order buyerOrder                                   = 2;
    OrderConfirmation vendorOrderConfirmation          = 3;
    repeated OrderFulfillment vendorOrderFulfillment   = 4;
    OrderCompletion buyerOrderCompletion               = 5;
    Dispute dispute                                    = 6;
    DisputeResolution disputeResolution                = 7;
    Refund refund                                      = 8;
    repeated Signature signatures                      = 9;
}
```

Each section of the contract is signed by the appropriate party's identity key. For example, the vendor signs the `Listing` object while the buyer signs the `Order` object. As the order progresses through different states, new objects are appended to the contract along with their signatures. When a dispute is filed with a moderator, the contract is sent to the moderator, programmatically validated, and then marshaled to JSON for the moderator to read. The contract contains all the information a moderator needs to make a decision and doesn't provide any wiggle room for the buyer or vendor to attempt to manipulate the outcome. 

The Ricardian Contract structure is very extensible and allows virtually an unlimited number of contract types to be created.
