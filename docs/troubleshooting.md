# Troubleshooting

If you are having problems receiving messages, it is most likely a firewall issue. Try forwarding Port 4001. You can also disable any IPv6 firewall if your router has that option.

You can change the port in the config file in "Gateway" under "Addresses".

If the buyer says that they've paid but you haven't received the Bitcoin, go to Settings->Advanced->Reload transaction and you should see confirmation of the Bitcoin transactions within 10-15 minutes.

We're working on several known issues in the Version 2 Beta, if you run into a problem, please let us know the following:
1. your Operating System
2. your firewall, if any
3. if you're running through a VPN
4. your anti-virus software, if any.
5. if you used the installer or installed from source using the command line

The two major issues we're tracking, and some simple fixes you can try:

A. messages are not received. This will mean you don't see a new order at all. Your node may be unable to receive messages, or it may be an issue with the sender. 

You can try turning your OB app off and back on to see if missing messages are detected and downloaded. If you installed from source, make sure to turn your server off and back on.

B. transactions are out of sync. This means you can see orders, but they show as Awaiting Payment. An order can be Awaiting Payment if the buyer just never paid (that happens), but it could also mean you didn't receive the data that the payment went through.

You can try to resolve this by going into your settings, on the advanced tab, and clicking Reload Transactions. If you do this, wait 15 minutes and check your transactions again. 

If you run into issues like these, please let us know, we would like to talk to you to gather as much information as possible to help us track down what the causes of these two bugs are.

If you've had any issues with a buyer making a payment that has not shown up for the order one thing you should do to debug is visit http://canyouseeme.org/

Enter port 4001 click check port (with openbazaar running). If it does not say `success` you'll likely need to forward the port 4001 in your router. 

You can click the `Reload Transactions` button in settings to resync the blockchain and detect missing transactions but the issue is the blockchain has synced past your transaction because you didn't detect the order in a timely manner due to your firewall settings.