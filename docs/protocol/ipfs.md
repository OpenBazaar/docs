# IPFS

## Introduction to IPFS
<a href="https://ipfs.io/">IPFS</a> stands for InterPlanetary File System. It is a hypermedia distribution protocol which forms the core the OpenBazaar network. It uses a Kademlia DHT, to route downloaders to those seeding files. What makes it unique is how IPFS serializes the data to create a cryptographically authenticated data structure known as a *Merkle* *DAG*. 

## OpenBazaar fork of IPFS

The OpenBazaar daemon uses a fork of the [go-ipfs](www.github.com/ipfs/go-ipfs) repository so you will not be able to access non-OpenBazaar related content when connected to the OpenBazaar network. There are different protocol strings to segregate the OpenBazaar network from the main IPFS network and an increased TTL on certain types of DHT data. Offline stores are seeded on the network for a week rather than the default 24 hours. You can find the full diff in the README of the forked [repo](https://github.com/OpenBazaar/go-ipfs). The fork is bundled in the vendor package and will be used automatically when you compile and run the server. Note that you will still see github.com/ipfs/go-ipfs import statements instead of github.com/OpenBazaar/go-ipfs despite the package being a fork. This is done to avoid a major refactor of import statements and make rebasing IPFS much easier.

[Note: much of the following description of IPFS is taken verbatim from <a href="http://whatdoesthequantsay.com/2015/09/13/ipfs-introduction-by-example">Christian Lundkvist</a> since he did such a great job]

## IPFS Objects

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
 
### Small Files
Small files (<256 kB) are represented as an IPFS object with the file data in the `Data` field and no `Links`. For example, a text file that says "Hello World" would look like this:
```
{
  "Links": [],
  "Data": "\u0008\u0002\u0012\rHello World!\n\u0018\r"
}
```
And in a more visual form:
<img src="https://imgur.com/download/m2VwTzR">

### Large Files
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

### Versioning
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

And since our peerId is the SHA256 hash of our Ed25519 public key, anyone can fetch the latest copy of our `root` hash from the DHT and validate the signature against our public key, which itself should hash to our `peerID`. 

In this manner, one only needs to know our `peerID` to download an authenticated copy of all of our store content. 

Using the IPNS protocol the above API call which fetched the listing could be rewritten as `/ipns/QmdHkAQeKJobghWES9exVUaqXCeMw8katQitnXDKWuKi1F/listings/coot-t-shirt.json` where `QmfHTiFpqLDAVj29Nf7LrfUFfz4envqArY4Gv7CvbyDcPt` is our `peerID`.

And by using other naming protocols such as <a href="https://blockstack.org/">Blockstack</a> we can cryptographically map a user's `peerID`, which is a rather ugly looking series of numbers and letters, to a more human readable username such as `@UrbanArt`. Thus one only needs to know the human readable username to download user content in a cryptographically secure manner. 