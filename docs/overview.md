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
