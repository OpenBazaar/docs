<h1>OpenBazaar Developer Reference</h1>

## What is OpenBazaar?
OpenBazaar is a new way to buy and sell goods and services online. By running a program on your computer, you can connect directly to other users in the OpenBazaar network and trade with them. The network isn't controlled by a company or organization. OpenBazaar is a decentralised peer-to-peer network, which means there are no listing fees and the marketplace is censorship-resistant.

Goods and services are bought and sold on OpenBazaar using Bitcoin, a digital cryptocurrency that is decentralised and censorship-resistant. Transaction fees on the Bitcoin network are very cheap.

**Bitcoin and OpenBazaar together make online commerce cheaper and more free than ever before.**

Right now, online commerce means using centralized services. eBay, Amazon, Alibaba and others have restrictive policies and charge fees for listing and selling goods. They only accept forms of payment that cost both buyers and sellers money, such as credit cards or PayPal/Alipay. They require personal information, which can lead to it being stolen or even sold to others for advertising or worse. Buyers and sellers aren’t always free to exchange goods and services with each other, as companies and governments censor entire categories of trade.

OpenBazaar is a different approach to online commerce. It puts the power back in the users’ hands. Instead of buyers and sellers going through a centralized service, OpenBazaar connects them directly. Because there is no one in the middle of the transactions, there are no fees, no one can censor transactions, and you only reveal the personal information that you choose.

This project is open source, which means the code is publicly available for people to review or audit, as well as contribute code to make it better.

## About this documentation
This developer reference aims to provide you with a technical understanding of how OpenBazaar works as well as document, in detail, the OpenBazaar protocol specification. The reference implementation of the OpenBazaar protocol is <a href="https://github.com/OpenBazaar/openbazaar-go">openbazaar-go</a>, server daemon written in the <a href="https://golang.org/">Go</a> programming language. 

While the openbazaar-go API is not part of the formal protocol spec, it is also documented here along with a number of examples to help you start building apps on top of the protocol. 

## Running OpenBazaar
The reference implementation is technically two separate applications ― a *daemon* which silently runs in the background and a user-interface which communicates with the running daemon. The *daemon* is the application which does all of the heavy-lifting. Among other things, it maintains peer to peer network connections, serves a user's product listings to other peers when requested, processes incoming orders and bitcoin transactions, and fetches content from other peers when you ask it to. 

The user interface, or client, runs in a packaged Chromium browser (see <a href="http://electron.atom.io/">electron</a>) and communicates with the daemon via an API.

A typical install of OpenBazaar bundles both the openbazaar-go daemon and user interface into a single application and launches both at the same time. From the user's perspective, it appears to be a single application.

The reason for this modular design is to make it easy for power users to run the openbazaar-go daemon on a separate device, perhaps a server, and keep it running 24/7 if they desire. In this configuration a user only needs the client installed on their computer and can connect it to the daemon via the internet. 

If you have not already done so, we suggest you download OpenBazaar and check it out. You can download the full (bundled) application from <a href="https://openbazaar.org/download.html">openbazaar.org</a> or install the <a href="https://github.com/OpenBazaar/openbazaar-go">daemon</a> and <a href="https://github.com/OpenBazaar/openbazaar-desktop">client</a> from source from their various Github repos. Instructions for installing from source can be found in each repo.  

## Contributing
OpenBazaar is an open source project and we welcome contributions of all types. If you are a developer feel free to submit issues or pull requests to both the openbazaar-go and openbazaar-desktop repos. If you want to contribute to this documentation, it can be found <a href="https://github.com/OpenBazaar/docs">on github</a>. We welcome you to file issues and submit pull requests to help us improve this documentation.
