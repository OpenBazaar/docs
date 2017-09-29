# Bitcoin

The early release of OpenBazaar 2.0 will only accept the cryptocurrency Bitcoin. There is a Shapeshift integration so you can still purchase items on OpenBazaar with other cryptocurrencies via Shapeshift. There are plans to facilitate direct purchases with other cryptocurrencies in the near future.

Bitcoin is the oldest and most secure cryptocurrency. It aligns with the goals of OpenBazaar as it is decentralized, censorship resistant and does not rely on trusted third parties.

Bitcoin enables a "trustless" escrow system using multi-signature transactions where no one party has complete control of the funds. Mainstream payment systems like Paypal and credit cards rely on the payment provider for arbitration in the case of a dispute. With OpenBazaar, a free market for arbitration is created where the participants can agree on a particular moderator. This moderator only has partial control of the funds in escrow.

The activation of SegWit in Bitcoin Core should decrease transaction fees in the short term but there are multiple cryptocurrencies that can offer additional or superior features such as lower transaction fees, shorter block times or stronger privacy.

In theory, the OpenBazaar protocol could work with any cryptocurrency, not just Bitcoin. In practice, there are certain features a cryptocurrency must have to be added seamlessly to the OpenBazaar protocol. Multi-signature transactions must be enabled and a UTXO model is preferential to the account model. In summary, cryptocurrencies that are a close derivative to Bitcoin are stronger candidates for integrations in the short term.

## Multi-sig scripts
If you'd like to learn more about how Bitcoin works we'd suggest reading the <a href="https://bitcoin.org/en/developer-guide">Bitcoin Developer Guide</a> or [Mastering Bitcoin](https://www.amazon.com/Mastering-Bitcoin-Programming-Open-Blockchain/dp/1491954388/) by Andreas Antonopoulos. However, we can provide a quick overview of how the escrow system works. 

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

## Bitcoin Wallets
The OpenBazaar protocol specification has nothing to say about which Bitcoin wallet should be used with the protocol. To improve the user experience the reference implementation comes bundled with a built-in wallet. The default wallet implements something call Simplified Payment Verification (SPV) which provides strong cryptographic validation of incoming Bitcoin transactions while using very little of the computer's resources. The drawback to SPV mode is it leaks enough private data to allow potential attackers to figure out which transactions came from the wallet. That information by itself doesn't say who the *owner* of the wallet is, though other investigative techniques might provide that information. 

For this reason, there is a setting in the openbazaar-go config file that allows a user to use bitcoind (a full Bitcoin implementation) with openbazaar-go. Bitcoind is a very heavyweight software and is typically only used by power users, but it does a much better job than SPV at providing transactional privacy. 

## Bitcoin Fees

How are Bitcoin fees calculated?

Bitcoin transaction fees are estimated using the 21 Bitcoin fees API. More details [here](https://bitcoinfees.21.co/api).