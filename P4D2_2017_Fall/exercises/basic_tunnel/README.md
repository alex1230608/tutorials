# Implementing Basic Tunneling

## Introduction

In this exercise, we will add support for a basic tunneling protocol
to the IP router that you completed in the previous assignment. To do so,
we will define a new header type to encapsulate the IP packet and 
modify the switch to perform routing using our new header.

The new header type will contain a protocol ID, which indicates the type
of packet being encapsulated, along with a destination ID to be used for
routing.

> **Spoiler alert:** There is a reference solution in the `solution`
> sub-directory. Feel free to compare your implementation to the
> reference.

## Step 1: Run the (incomplete) starter code

The starter code for this assignment is in a file called `basic_tunnel.p4` 
and is simply the solution to the IP router from the previous exercise.

Let's first compile this code to and send a packet between two end hosts
to ensure that the IP routing is working as expected.

1. In your shell, run:
   ```bash
   ./run.sh
   ```
   This will:
   * compile `basic_tunnel.p4`, and
   * start a Mininet instance with three switches (`s1`, `s2`, `s3`)
     configured in a triangle, each connected to one host (`h1`, `h2`,
     and `h3`).
   * The hosts are assigned IPs of `10.0.1.1`, `10.0.2.2`, and `10.0.3.3`.

2. You should now see a Mininet command prompt. Open two terminals
for `h1` and `h2`, respectively:
   ```bash
   mininet> xterm h1 h2
   ```
3. Each host includes a small Python-based messaging client and
server. In `h2`'s xterm, start the server:
   ```bash
   ./receive.py
   ```
4. In `h1`'s xterm, send a message to `h2`:
   ```bash
   ./send.py 10.0.2.2 "P4 is cool"
   ```
   The packet should be received at `h2`. If you examine the received
   packet you should see that is consists of an Ethernet header, an IP
   header, and the message. If you change the destination IP address
   (e.g. try to send to `10.0.3.3`) then the message should not be 
   received by h2.
5. Type `exit` or `Ctrl-D` to leave each xterm and the Mininet command line.

Each switch is forwarding based on the destination IP address. Your
job is to change the switch functionality so that they instead decide
the destination port using our new tunnel header.

### A note about the control plane

A P4 program defines a packet-processing pipeline, but the rules
within each table are inserted by the control plane. When a rule
matches a packet, its action is invoked with parameters supplied by
the control plane as part of the rule.

In this exercise, you will need to add a couple of static control plane
rules for the tunneling protocol. As part of bringing up the Mininet 
instance, the `run.sh` script will install packet-processing rules in
the tables of each switch. These are defined in the `sX-commands.txt`
files, where `X` corresponds to the switch number.

**Important:** A P4 program also defines the interface between the
switch pipeline and control plane. The commands in the files
`sX-commands.txt` refer to specific tables, keys, and actions by name,
and any changes in the P4 program that add or rename tables, keys, or
actions will need to be reflected in these command files.

## Step 2: Implement Basic Tunneling

The `basic_tunnel.p4` file contains an implementation of a basic IP router.
It also contains comments marked with `TODO` which indicate the functionality
that you need to implement. A complete implementation of the `basic_tunnel.p4`
switch will be able to forward based on the contents of a custom encapsulation
header as well as perform normal IP forwarding if the encapsulation header
does not exist in the packet.

Your job will be to do the following:

1. **TODO:** Add a new header type called `myTunnel_t` that contains two 16-bit fields: `proto_id` and `dst_id`.
2. **TODO:** Add a `myTunnel_t` header to the `headers` struct.
2. **TODO:** Update the parser to extract either the `myTunnel` header or `ipv4` header based on the `etherType` field in the Ethernet header. The parser should also extract the `ipv4` header after the `myTunnel` header if `proto_id` == `TYPE_IPV4` (i.e. 0x0800).
3. **TODO:** Define a new action called `myTunnel_forward` that simply sets the egress port (i.e. `egress_spec` field of the `standard_metadata` bus) to the port number provided by the control plane.
4. **TODO:** Define a new table called `myTunnel_exact` that perfoms an exact match on the `dst_id` field of the `myTunnel` header. This table should invoke either the `myTunnel_forward` action if the there is a match in the table and it should invoke the `drop` action otherwise.
5. **TODO:** Update the `apply` statement in the `MyIngress` control block to apply your newly defined `myTunnel_exact` table if the `myTunnel` header is valid. Otherwise, invoke the `ipv4_lpm` table if the `ipv4` header is valid.
6. **TODO:** Update the deparser to emit the `ethernet`, then `myTunnel`, then `ipv4` headers. Remember that the deparser will only emit a header if it is valid. A header's implicit validity bit is set by the parser upon extraction. So there is no need to check header validity here.
7. **TODO:** Add static rules for your newly defined table so that the switches will forward correctly for each possible value of `dst_id`. See the diagram below for the topology's port configuration as well as how we will assign IDs to hosts. For this step you will need to add your forwarding rules to the `sX-commands.txt` files.

![topology](./topo.pdf)

## Step 3: Run your solution

Follow the instructions from Step 1. This time when you send a packet from
`h1` to `h2` try using the following command to send a packet that uses
our new `myTunnel` header.
```bash
./send.py 10.0.2.2 "P4 is cool" --dst_id 2
```

You should see a packet arrive at `h2` which contains the `MyTunnel` header.
Also note that changing the destination IP address will not prevent the packet
from arriving at `h2`. This is because the switch is no longer using the IP header for routing when the `MyTunnel` header is in the packet.

> Python Scapy does not natively support the `myTunnel` header 
> type so we have provided a file called `myTunnel_header.py` which 
> adds support to Scapy for our new custom header. Feel free to inspect
> this file if you are interested in learning how to do this.

### Food for thought

To make this tunneling exercise a bit more interesting (and realistic)
how might you change the P4 code to have the switches add the `myTunnel`
header to an IP packet upon ingress to the network and then remove the
`myTunnel` header as the packet leaves to the network to an end host?

Hints:

 - The ingress switch will need to map the destination IP address to the corresponding `dst_id` for the `myTunnel` header. Also remember to set explicitly set the validity bit for the `myTunnel` header so that it can be emitted by the deparser.
 - The egress switch will need to remove the `myTunnel` header from the packet after looking up the appropriate output port using the `dst_id` field.

### Troubleshooting

There are several problems that might manifest as you develop your program:

1. `basic_tunnel.p4` might fail to compile. In this case, `run.sh` will
report the error emitted from the compiler and halt.

2. `basic_tunnel.p4` might compile but fail to support the control plane
rules in the `s1-commands.txt` through `s3-command.txt` files that
`run.sh` tries to install using the Bmv2 CLI. In this case, `run.sh`
will report these errors to `stderr`. Use these error messages to fix
your `basic_tunnel.p4` implementation or forwarding rules.

3. `basic_tunnel.p4` might compile, and the control plane rules might be
installed, but the switch might not process packets in the desired
way. The `build/logs/<switch-name>.log` files contain detailed logs
that describing how each switch processes each packet. The output is
detailed and can help pinpoint logic errors in your implementation.

#### Cleaning up Mininet

In the latter two cases above, `run.sh` may leave a Mininet instance
running in the background. Use the following command to clean up
these instances:

```bash
mn -c
```

## Next Steps

Congratulations, your implementation works! Move onto the next assignment
[p4runtime](../p4runtime)!

