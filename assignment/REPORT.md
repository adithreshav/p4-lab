# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **1484**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

     I created the tunnel by changing the myTunnel_exact entry for key 3, from port 3 to port 1 using the sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1" command. 
     
     After sending a single tunneled packet with --tunnel 3, I used the sh grep -c "Processing packet received" logs/s1.txt logs/s2.txt and sh sleep 5; grep -c "Processing packet received" logs/s1.txt logs/s2.txt commands which shows the count go from 73 (s1) and 70 (s2) to 150 and 148 because the packet is looping between the 2 switches. s1 had a little higher count fure to the "h1 python3 send.py 10.84.0.3 "loop" --tunnel 3" command.

     The TTL didnt stop an IPv4 loop because nothing in MyIngress or MyEgress checks the value. The only drops in the program are the table miss ((default_action = drop()) or a packet with no valid tunnel or IPv4 header. In ipv4_forward theres a line "hdr.ipv4.ttl = hdr.ipv4.ttl - 1;". Based on that, when it reaches 0 and it has to minus 1, it will wraparound and reset to 255 since ttl is a 8-bit field. 

## 2. Design: the hop limit

     Field size: 8 bits. The reason I chose 8 bits is because 255 hops is way more than enough for this netwok and the same range as IPv4's TTL. It is also one byte which Scapy's ByteField handles directly. The position I placed it is after dst_id. This is for backward compatibily because when a old switch readds only the first 4 byte, it will still be correct.

     Starting value: I choose 8 because it will give mmore room to the network to grow. Smaller value means it will be dropped earlier eventhough currently there is only 2 switches that it crosses. 

     Where it's decremented: In ingress, before the myTunnel_exact lookup. If the hop limit is 0, the packet will be dropped. Otherwise, it will decrement by 1 and will be forwarded to the next hop. It is decremented at the ingress so that the packet is dropped before it takes queue space if the hop limit is 0 already.

     What else changed: The host changed, the send.py and receive.py files define the header. Therefore, they both needed the new ByteField, in the same position.

## 3. Implementation

     What changed
     An 8-bit hop limit is added to myTunnel_t in tunnel.p4, and in the ingress, the hop limit is checked and decrement by one each time until it reaches 0 and will be dropped. 

     The ByteField added in both hosts' MyTunnel classes. send.py now takes a new paramater which is --hops and the hop limit is printed by receive.py


## 4. Evidence: it works

     Tunnelled delivery (delivery.txt): both packets arrived, at h2 and h3, with hop_limit=6, TTL=64. As I mentioned in delivery.txt, the hop limit is 6 because each switch on the tunnel path checks the hop limit and decrements it by 1. Since it goes to s1 and s2 it decrements by 2. The TTL is still 64 because the tunneled packets go through myTunnel_exact which only sets the output port.

     IPv4 still works (delivery.txt): The ping got both replies with TTL 62 because ipv4_lpm which has actions = { ipv4_forward; drop; }, decrements the TTL. 
     
     The loop ends (loop_after.txt): As shown in loop_after.txt, the packet was forwarded 8 times and dropped at s1 on the fifth time because hop_limit == 0 became true at packet [6.0]. Besides the first 2 packets in s1.txt that was is not related to the hop limit, s1 has 5 passes and s2 has 4, exactly as the design. The counts didn’t change after 5 seconds because the packet is dropped and no loop occured.

## 5. Evaluation

     a) The mixed rollout (mixed.txt): 
        I predicted that s2 will ignore the hop limit part because it is using the old file which doesnt have the hop limit update. In my opinion, it will still forward the packet to h2 because s2 will read the first 4 bytes as the tunnel header which is proto_id and the dst_id. At h2, the hop limit will be 7 instead of 6 because the old tunnel_v1.p4 doesnt have the decrement code implemented. For the loop, I predict that it will end but after twice the amount of hops because s1 will decrement the limit but s2 wouldnt and on the 9th time s1 recieves the packet it will drop.
        
        After running the commands, the result is that the packet did arrive to h2 with the right length, addresses, and payload, and the hop limit as predicted. However the IPv4 header now says protocol 13 instead of UDP. This is because the old version (which s2 is using) parsed the IPv4 starting one byte early at the hop limit byte. 

        Byte by byte: 
        ffff ffff ffff - Ethernet destination (broadcast)
        0000 0000 0001 - Ethernet src (h1)	
        1212 - etherType	(tunnel)
        0800 - proto_id	(IPv4 next)	
        0002 - dst_id	(exit 2)
        07 - hop_limit	(7)	s1 decremented 8 → 7; old s2 didn’t. As I predicted
        45 - IPv4 version/IHL	(4, 5)
        00 - TOS (0)
        0021 - total length (33)	20 + 8 + 5 for "mixed"
        0001 - ID	(1)
        0000 - flags/fragment	(0)	
        40 - TTL (64) - untouched by the tunnel path
        0d - protocol (13)	should be 11 (17, UDP). Corrupted due to s2 taking the IPv4 one byte early which is the hop limit byte
        6620 - IPv4 checksum	(0x6620)
        0a54 0001 - src IP	(10.84.0.1)	
        0a54 0003 - dst IP	(10.84.0.3)	
        10e1 10e1 000d 7f97 - UDP	(ports 4321, length 13)	
        6d69 7865 64 - payload	(mixed)	

        For the loop, it happened as predicted the condition "hdr.myTunnel.hop_limit == 0" in s1.txt was false 9 times before it became true and dropped.

     b) Propose a safe way to roll out a header change across a network of switches that can't all be upgraded at once. Compare at least two options (for example a version field, a new EtherType, or ordering fields so old switches still parse them).

     Approach 1: Add a version field to the header. The first field will mention which format it uses. A switch will read the version, then parse the rest accordingly but if it sees a version that it doesn't know, it can either drop it or forward by dst_id without misreading it. The pros is that only one EtherType is needed to cover all versions and switches will handles the unknown versions safely. The cons to it is that an extra header will be added.

     Approach 2: The new header will get its own EtherType. The pros is that it works well and when an old switch doesn't recognize an EtherType, it will stop parsing after Ethernet and drop the packetm which is better than a silent corruption. The new switches will also be able to support both EtherTypes during the transition. The cons is that senders must not use the new format until the whole path is upgraded to avoid the packet from dropping. 

     Approach 2: 

     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's check_paths()): what can each catch that the other can't? Why do real networks use both?

     The loop protection in the data plane can catch changes made directly such as C1 where changes are made directly with p4cli without going through any controller. The check_paths() wont be able to detect that because it works on the controller's own copy of the forwarding state which is the entries it planned and installed. The data plane loop protection can also catch bugs and faults below the control plane. The control plane assumes the switch does exactly what the tables say. If the P4 program has a bug, or the hardware corrupts a table entry, or a switch forwards wrongly for any other reason, the controller’s view is still correct but the switch’s behaviour isn’t. Besides that, the hop limit in the data plane loop protection catches transient loops where routes are updated one at a time. That mismatch can form a loop eventhough both the old and new configuration pass the control plane's check. 
     
     As for the control plane, it can catch a loop before any traffic is sent, which means no bandwidth is needed. For the hop limit, every loop still need to make 9 passes before it is dropped. It also tells which entry is wrong instead of silently dropping like the hop limit. The control plane also can catch the difference between a loop or a long path because it can see the forwarding state, the hop limit only drops based on amount of hops. Loop protection in the control plane also can catch faults such as black holes. This is because it can see the path step by step and check for switch with no entry for the destination and flag it which the hop limit in loop protection cant do.
     
     d) What would this change cost on a hardware switch running at line rate? 
        It requires a higher bandwidth because there is an extra byte on every tunneled packet which reduces how many packets per second fit. It will also affect the parser because a slightly longer header have to be extracted. There are 2 new operations as well which is the comparison to 0 for the hop limit, and the decrement. 

## AI use

     AI Use (Level 3): Use Claude to understand the lab instructions and further understand the output of certain commands. For example, in C5 to understand the logs to calculate how many times the packets pass through the switches. I also used AI for extra commands to double check certain outcomes. 

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
