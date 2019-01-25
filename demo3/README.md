# Compiling

See [README-using-bmv2.md](../README-using-bmv2.md) for some things
that are common across different P4 programs executed using bmv2.

To compile the P4_16 version of the code:

    p4c --target bmv2 --arch v1model demo3.p4_16.p4

To compile the P4_14 version of the code:

    p4c --std p4-14 --target bmv2 --arch v1model demo3.p4_14.p4

The .dot and .png files in the subdirectory 'graphs' were created with
the p4c-graphs program, which is also installed when you build and
install p4c-bm2-ss:

     p4c-graphs -I $HOME/p4c/p4include demo3.p4_16.p4

The '-I' option is only necessary if you did _not_ install the P4
compiler in your system-wide /usr/local/bin directory.


# Running

To run the behavioral model with 8 ports numbered 0 through 7:

    sudo simple_switch --log-console -i 0@veth2 -i 1@veth4 -i 2@veth6 -i 3@veth8 -i 4@veth10 -i 5@veth12 -i 6@veth14 -i 7@veth16 demo3.p4_16.json

To run CLI for controlling and examining simple_switch's table
contents:

    simple_switch_CLI

General syntax for table_add commands at simple_switch_CLI prompt:

    RuntimeCmd: help table_add
    Add entry to a match table: table_add <table name> <action name> <match fields> => <action parameters> [priority]

----------------------------------------------------------------------
simple_switch_CLI commands for demo3 program
----------------------------------------------------------------------

    # These should be unnecessary for P4_16 program, which defines
    # these default actions with default_action assignments in its
    # table definitions.
    table_set_default compute_ipv4_hashes compute_lkp_ipv4_hash
    table_set_default ipv4_da_lpm my_drop
    table_set_default mac_da my_drop
    table_set_default send_frame my_drop

Add both sets of entries below:

    # For P4_16 program, set_l2ptr action name for table ipv4_da_lpm
    # is changed to set_l2ptr_with_stat.
    
    # what these three do: 
    # what u see, actually are : 
    # 1. when 10.1.0.1 hit, in L2, we give it a ptr of 58
    # 1.1 which means that 58 will be used in next table(mac_da table)
    # 1.2 next table (mac_da table) is used for get some info about: 
    # 1.3 1. bridge domain, 2. dst_mac(eg: PC2 mac:02:13:57...) 3. intf(eg: egress port is 2)
    # 2. when 58 is hit in mac_da table
    # 2.1 we can get info that given by Control Panel previously
    # 2.2 we put info we get from CP into metadata; like 
    table_add ipv4_da_lpm set_l2ptr_with_stat 10.1.0.1/32 => 58
    # 2.3 meta.fwd_metadata.out_db = 9;//here 9 is just we saw in following
    table_add mac_da set_bd_dmac_intf 58 => 9 02:13:57:ab:cd:ef 2
    # 3. ok, last we will see how we rewrite the pkt's src mac
    table_add send_frame rewrite_mac 9 => 00:11:22:33:44:55

    # what these three do:
    # actually the same as above, except for this one is for pkt who 
    # looking for 10.1.0.200
    table_add ipv4_da_lpm set_l2ptr_with_stat 10.1.0.200/32 => 81
    table_add mac_da set_bd_dmac_intf 81 => 15 08:de:ad:be:ef:00 1
    table_add send_frame rewrite_mac 15 => ca:fe:ba:be:d0:0d

    # For P4_14 program, use this
    table_add ipv4_da_lpm set_l2ptr 10.1.0.1/32 => 58
    table_add mac_da set_bd_dmac_intf 58 => 9 02:13:57:ab:cd:ef 2
    table_add send_frame rewrite_mac 9 => 00:11:22:33:44:55

    table_add ipv4_da_lpm set_l2ptr 10.1.0.200/32 => 81
    table_add mac_da set_bd_dmac_intf 81 => 15 08:de:ad:be:ef:00 1
    table_add send_frame rewrite_mac 15 => ca:fe:ba:be:d0:0d

The entries above with action set_l2ptr on table ipv4_da_lpm work
exactly as they did before.  They avoid needing to do a lookup in the
new ecmp_group table.

Here is a first try at using the ecmp_group and ecmp_path tables for
forwarding packets.  It assumes that the table entries above are
already added.

I want all packets that match 11.1.0.1/32 to go through a 3-way ECMP
table to output ports 1, 2, and 3, where ports 1 and 2 reuse the
mac_da table entries added above.

tt: what he means: when u send pkt to 11.1.0.1(not 10.1.0.1 nor 10.1.0.200, which 
are add above by hand), the P4 switch will use ECMP to forward this pkt(to 11.1.0.1)

    # mac_da and send_frame table entries for output port 3
    # tt watch out: here we don't see anything like: set_l2ptr...things 
    # so how pkt reach to port 3? aka: when will l2ptr will set to 101, (so 
    # that we can send pkt to port3, by point l2 action with index 101)
    # tdo: 0125: see `table_add ecmp_path set_l2ptr 67 2 => 101` that's where
    # the 101(l2ptr) was set (so that we can send pkt to port 3)
    table_add mac_da set_bd_dmac_intf 101 => 22 08:de:ad:be:ef:00 3
    table_add send_frame rewrite_mac 22 => ca:fe:ba:be:d0:0d
    
    # LPM entry pointing at ecmp group idx 67.
    # Then ecmp_group entry for ecmp group idx 67 that returns num_paths=3.
    # Then 3 ecmp_path entries with ecmp group idx 67, and
    # ecmp_path_selector values in the range [0,2], each giving a
    # result with a different l2ptr value.

    table_add ipv4_da_lpm set_ecmp_group_idx 11.1.0.1/32 => 67
    table_add ecmp_group set_ecmp_path_idx 67 => 3
    table_add ecmp_path set_l2ptr 67 0 => 81
    table_add ecmp_path set_l2ptr 67 1 => 58
    table_add ecmp_path set_l2ptr 67 2 => 101


----------------------------------------------------------------------
scapy session for sending packets
----------------------------------------------------------------------
I believe we must run scapy as root for it to have permission to send
packets on veth interfaces.

    sudo scapy

    fwd_to_p1=Ether() / IP(dst='10.1.0.200') / TCP(sport=5793, dport=80) / Raw('The quick brown fox jumped over the lazy dog.')
    fwd_to_p2=Ether() / IP(dst='10.1.0.1') / TCP(sport=5793, dport=80)
    drop_pkt1=Ether() / IP(dst='10.1.0.34') / TCP(sport=5793, dport=80)

    # Send packet at layer2, specifying interface
    sendp(fwd_to_p1, iface="veth2")
    sendp(fwd_to_p2, iface="veth2")
    sendp(drop_pkt1, iface="veth2")

    # For packets going to the ECMP group, vary the source IP address
    # so that each will likely get different hash values.

    # The hash value for this one caused it to go out port 2 in my
    # testing.
    fwd_to_ecmp_grp1=Ether() / IP(dst='11.1.0.1', src='1.2.3.4') / TCP(sport=5793, dport=80)
    # output port 1 for this packet
    fwd_to_ecmp_grp2=Ether() / IP(dst='11.1.0.1', src='1.2.3.5') / TCP(sport=5793, dport=80)
    # output port 3 for this packet
    fwd_to_ecmp_grp3=Ether() / IP(dst='11.1.0.1', src='1.2.3.7') / TCP(sport=5793, dport=80)

----------------------------------------


# Patterns

The example table entries and sample packet given above can be
generalized to the following 3 patterns.


## Pattern 1 - no ECMP

The first pattern is for packets that do not do ECMP at all, because
their longest prefix match lookup result directly gives an l2ptr.

If you send an input packet like this:

    input port: anything
    Ether() / IP(dst=<hdr.ipv4.dstAddr>, ttl=<ttl>)

and you create the following table entries:

    table_add ipv4_da_lpm set_l2ptr_with_stat <hdr.ipv4.dstAddr>/32 => <l2ptr>
    table_add mac_da set_bd_dmac_intf <l2ptr> => <out_bd> <dmac> <out_intf>
    table_add send_frame rewrite_mac <out_bd> => <smac>

then the P4 program should produce an output packet like the one
below, matching the input packet in every way except, except for the
fields explicitly mentioned:

    output port: <out_intf>
    Ether(src=<smac>, dst=<dmac>) / IP(dst=<hdr.ipv4.dstAddr>, ttl=<ttl>-1)



## Pattern 2 - no ECMP, but one level of indirection

The second pattern is for packets that go through one level of
indirection in the ecmp_group table, but skip over the ecmp_path
table.  This is useful to software for having many longest prefix
match entries point at a since ecmp_group table entry, but by having
the indirection, all of those prefixes can be updated to a new output
port and source MAC address with a single write to the ecmp_path
table.

The only differences between pattern 2 and pattern 1 are in table
ipv4_da_lpm and ecmp_group.  mac_da and send_frame are the same as
before.

If you send an input packet like this:

    same as pattern 1

and you create the following table entries:

    table_add ipv4_da_lpm set_ecmp_group_idx <hdr.ipv4.dstAddr>/32 => <ecmp_group_idx>
    table_add ecmp_group set_l2ptr <ecmp_group_idx> => <l2ptr>
    same as pattern 1 from table mac_da onwards



## Pattern 3 - full ECMP

Software should use this for equal cost multipath routing,
i.e. multiple shortest paths to the same destination, over which
traffic should be load balanced, based upon a hash calculated from
some packet header field values specified in action
compute_lkp_ipv4_hash.

If you send an input packet like this:

    same as pattern 1

and you create the following table entries:

    table_add ipv4_da_lpm set_ecmp_group_idx <hdr.ipv4.dstAddr>/32 => <ecmp_group_idx>
    table_add ecmp_group set_ecmp_path_idx <ecmp_group_idx> => <num_paths>
    table_add ecmp_path set_l2ptr <ecmp_group_idx> <ecmp_path_selector> => <l2ptr>
    same as pattern 1 from table mac_da onwards

NOTE: <ecmp_path_selector> is a hash calculated from the packet, then
modulo'd by <num_paths>, so it can be any number in the range [0,
<num_paths>-1].  For this pattern, it would be good to install
<num_paths> entries in the ecmp_path table, and for each one of those
<l2ptr> values, there should be corresponding entries in the mac_da
and send_frame tables.  It is OK to have multiple ecmp_path entries
with the same <l2ptr> value -- this is normal when software creates
such tables, especially across different <ecmp_group_idx> values.

When checking the output packet, it is correct to receive an output
packet for any of the <num_paths> possible paths.  If the test
environment wants to narrow it down to 1 possible output packet, it
must do the same hash function that the data path code is doing.
