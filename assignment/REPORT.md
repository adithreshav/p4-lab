# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **NNNN**

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

<!-- An 8-bit hop limit is added to myTunnel_t in tunnel.p4, and in the ingress, the hop limit is checked and decrement by one each time until it reaches 0 and will be dropped. 

The ByteField added in both hosts' MyTunnel classes
Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
