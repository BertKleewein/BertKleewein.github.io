# Using Wireshark to intercept and decode AMQP sent to Azure IoT Hub

Wireshark is a powerful tool, but it's hard to use, especially if you don't eat, sleep, and breathe network protocols.
I recently figuerd out how to use Wireshark to capture AMQP traffic being sent to Azure IoT Hub.
More importantly, I figured out how to configure Wireshark to decrypt and display the raw AMQP packets.
I've wanted to know how to do this for a long time, so I'm writing this article to explain how I did it.

# But, why?

I needed to track down a bug in the [Azure IoT Service SDK for Python](http://github.com/Azure/azure-iot-sdk-python/service).
Unfortunately for me, the "guts" of the AMQP code is written in C.
I could have compiled this library with debug information and run my test code under GDB, but this was a hassle.
Instead, I decided to see if I could find the bug using Wireshark.

# Breaking security, but not really.

To look at the data, you're going to do a "man in the middle" (MITM) attack against yourself.
But, I'm not explaining how to "hack" anything.
I'm not explaining how to defeat any security.
These instructions require you to have root access to the machine you're using.
They also require you to modify some code so it trusts this "man in the middle."
In doing so, you're also giving Wireshark access to the necessary decryption keys.

The connection between your code and the MITM is slightly weaker, but the connection between the MITM and Azure IoT Hub is just as strong as it always is.
Since the MITM is running on your computer, all traffic that goes onto the internet is completely secure.

# Un-editied content begins here

Caveats:
* These instructions _will not_ work for Windows.
* They _should_ work with MQTT (with adjustments).
* This has only been tested with symmetric key authentication.
* I'm sure a similar pattern could be used for X509 authentication, but the details would be different.

# Step 1: Install mitmproxy from http://mitmproxy.org.
Download the latest version instead of using the one that might be in your repository's package manager.
The easiest way is to use instructions on the site.

I did this using bash to put binaries into the current directory.

```
curl https://snapshots.mitmproxy.org/7.0.4/mitmproxy-7.0.4-linux.tar.gz
tar -xf mitmproxy-7.0.4-linux.tar.gz
```

# Step 2: Install wireshark.

I recommend downloading the latest version instead of the one used by your distro's package manager.
The version I got from Ubuntu 18.04's package manager was old enough that TLS options are different (this is important), so I used the stable release PPA to install 3.4.8.

```
sudo add-apt-repository ppa:wireshark-dev/stable
sudo apt-get update
sudo apt install -y wireshark
```

# Step 3: Configure mitmproxy to act as a transparent proxy

An explanation of transparent proxying can be found at https://docs.mitmproxy.org/stable/concepts-modes/#transparent-proxy.

Since my proxy is on the same machine as my client, I set it up using custom routing (https://docs.mitmproxy.org/stable/concepts-modes/#b-custom-routing)

If you have a second machine to act as a router, it _might_ be easier to use the 'custom gateway' option (https://docs.mitmproxy.org/stable/concepts-modes/#a-custom-gateway)

This is what I did, based on https://docs.mitmproxy.org/stable/howto-transparent/.
Since I'm using IPV4, I didn't bother with any of the IPV6 instructions.
Also, since I'm using AMQP, I have different port numbers here.

The instructions talk about persisting across reboots.
I didn't do this.
Feel free if you're brave.

```
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Disable ICMP redirects.
sudo  sysctl -w net.ipv4.conf.all.send_redirects=0
```

If you're following along, I need to jump around the instructions.
Since I'm on one machine, I need to follow the instructions at https://docs.mitmproxy.org/stable/howto-transparent/#work-around-to-redirect-traffic-originating-from-the-machine-itself.

Next, we need to create a user account to run the proxy under.
This is necessary to prevent recursive proxying.
Below, we'll create a routing rule that effectively says "forward all AMQP traffic into the proxy, unless it comes from the `mitmproxyuser` account. In that case, send it to the correct destination."

I do this in a different window which will eventually run the proxy:
```
sudo useradd --create-home mitmproxyuser
sudo -u mitmproxyuser -H bash
```

This should give you a bash promt for the user named `mitmproxyuser`.


Some of the mitmproxy plugins require python and the `mitmproxy` module.
Install this.
(This is where I switched to my python virtual environment, but I'm not including details).
```
pip3 install --user mitmproxy
```

Finally, start the proxy.
This is from the `mitmproxy` install you did way up top, so you should be in the directory that you untar'ed the binaries into.
The IP address here is the IP of your hub instance.
You can get this using ping (e.g. `ping -4 my-hub-name.azure-devices.net`)
```
export SSLKEYLOGFILE="/home/mitmproxyuser/.mitmproxy/sslkeylogfile.txt"
./mitmproxy --mode transparent --set block_global=false --rawtcp --tcp-hosts "40.78.204.71"
```

Later on, we're going to use the `sslkeylogfile.txt` later when we configure wireshark.
This is the key to getting the wireshark decription working (pun intended).

Inside the proxy window, hit `E` to show events. This will help you see when the proxy handles traffic.

# Step 4: Set up port forwarding.

We're done with the window that's running the `mitmproxyuser` code.
Switch back to your other window and set up forwarding rules to redirect traffic from AMQP ports to the port that mitmproxy is going to use.

```
sudo iptables -t nat -A OUTPUT -p tcp -m owner ! --uid-owner mitmproxyuser --dport 5671:5672 -j REDIRECT --to-port 8080
```

# Step 4A: How to remove port forwarding

The rule we added in step 4 will persist across reboots.

To remove it,
1. use `sudo iptables -t nat -L OUTPUT` to see the routing rules.
2. You need the number of the rule you want to delete, so if it's the first rule in the list, you delete rule 1, the second rule is 2, and so on.
3. Delete the rule using `sudo iptables -t nat -d OUTPUT x` where x is the rule number.
4. Use `sudo iptables -t nat -L OUTPUT` to verify that it got deleted.)

Since this is a little convoluted and somewhat necessary, here is a copy/paste of a bash session:
```bash

# To delete the rule, first we list the rules for the OUTPUT chain in the nat routing table

(Python-3.7.9) bertk@bertk-hp:~$ sudo iptables -t nat -L OUTPUT
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere            !localhost/8          ADDRTYPE match dst-type LOCAL
REDIRECT   tcp  --  anywhere             anywhere             ! owner UID match mitmproxyuser tcp dpts:amqps:amqp redir ports 8080

# Since our rule is the second one in the list, it's rule #2. Delete this rule

(Python-3.7.9) bertk@bertk-hp:~$ sudo iptables -t nat -D OUTPUT 2

# Now, list the same rules again and make sure our rule is gone

(Python-3.7.9) bertk@bertk-hp:~$ sudo iptables -t nat -L OUTPUT
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere            !localhost/8          ADDRTYPE match dst-type LOCAL

```

# Step 5: Set the proxy CA as a truested CA

I'm not 100% sure this is necessary, but it doesn't hurt.
When we started the `mitmproxy` process, it created a number of certs.
We need to take the CA cert and set it as trusted by copying it into our local trust store.

These instructions are for Ubuntu/Debian.
Instructions for other systems can be found at https://docs.mitmproxy.org/stable/concepts-certificates/

First, go into the directory where `mitmproxy` put the certs.

```
cd /home/mitmproxyuser/.mitmprox
```

Then make a directory in a specific location to hold the cert.

```
sudo mkdir /usr/local/share/ca-certificates/extra
```

Copy the the CA cert into the directory we just created.
It needs to be in PEM format with a crt excention.
```
sudo cp mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/extra/mitmproxy-ca-cert.crt
```

Finally, add the cert to the store:
```
sudo update-ca-certificates
```

# Step 6: Add the CA cert to client creation

We're almost there.
If we launch our client app now, it should route through the proxy, but fail to authenticate.
This is because the client is trying to verify that the hostname in the cert exactly matches the hostname of the iothub instance.

We get around this by passing the cert into the SDK as a "CA Cert" or "Server Verification Cert".


If we're using a device SDK, there should be an optional parameter to pass this in when you're creating the device client.

Since I'm testing this with the service SDK, I need to edit the SDK to pass the CA cert into the TLS library.

For Python, edit `azure/iot/hub/iothub_amqp_client.py` and add the cert as a parameter named `verify` to the `JWTTokenAuth` constructor. This is the same cert that you added as a trusted CA above.

```python
        auth = uamqp.authentication.JWTTokenAuth(
            audience="https://" + hostname,
            uri="https://" + hostname,
            get_token=get_token,
            token_type=b"servicebus.windows.net:sastoken",
            http_proxy=proxy_settings,
            verify="/home/bertk/temp/mitmproxy-ca_cert.pem",  # ADD THIS PARRAMETER
        )
```

For Node, add the cert as `connectionParameters.ca` inside the `connect` method inside `common/transport/amqp/src/amqp.ts`.


```typescript
  connect(config: AmqpBaseTransportConfig, done: GenericAmqpBaseCallback<any>): void {

    let parsedUrl = urlParser.parse(config.uri);
    let connectionParameters: any = {};
    connectionParameters.ca = fs.readFileSync("/home/bertk/temp/mitmproxy-ca-cert.pem", "utf-8");  // ADD THIS LINE
```

You'll also need to add `import * as fs from 'fs';` at the top of the file.

Don't forget to run `npm run build` to transpile the typescript to javascript.

# Step 7: Profit

If this all works, you should be able to run your app, see it connect successfully, and also see the connection inside the proxy window.

# Step 8: Run Wireshark

1. Launch Wireshark and open Edit-\>Preferences.  In the tree-view on the left, go to Protocols->TLS.  If you don't see "TLS" listed under Protocols, it means your version of Wireshark is probably out of date.  Upgrade it.
2. Under Edit-\>Preferences-\>Protocols-\>TCP, make sure that "Allow subdissector to reassemble TCP streams" and "Reassemble out-of-order segments" are both selected.
3. For the "(Pre)-Master-Secret log filename" field, browse to find the `sslkeylogfile.txt` file that you specified up in Step 3.
4. Close the Preferences dialog
5. Start a capture. I select my primary NIC (`eno2` on my mchine) and set a capture filter of `host <hub-ip>` (I used `host 40.74.204.71` for my hub).  The blue shark-fin icon on the left of the toolbar starts the capture.
6. Run the app and you should see your AMQP traffic in the wireshark window.  You should see some nodes that let you expand from TCP to TLS to AMQP and you should see decrypted data in the AMQP node.
7. If you want to only want to see the AMQP nodes without the TCP and TLS wrappers, you can enter `amqp` (in lowercase) in the "display filter" field.


