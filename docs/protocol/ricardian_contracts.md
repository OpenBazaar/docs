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