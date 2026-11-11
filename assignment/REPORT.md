# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **NNNN**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop
<!-- How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. -->
By modifying the entry 2 which suppose to forward the frame out of port 3, i change it to forward back to port 1 which is the entry port from s1, causing the packet loop between s1 and s2.
The switch log show spike of packet number increase within just 5 seconds from 500+ to 1500+ packets.
Decrementing the TLL wont stop the loop because there is not yet drop logic when TTL reach 0.

## 2. Design: the hop limit

<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->
8 bit for the field size which is the same with ipv4 TTL that allows up to 255 hops and also to keep the headers be a whole number of bytes. It is placed last in the header so the parser that work on proto_id no need to change

8 is choose as the starting value because it is sligthly longer than longest path in the lab topology. So a legitimate packets wont drop by mistake and it is small enough to discard a packet stuck in loop immediately. And i like the number 8 :D

It is decremented inside ingress which everytime the packet meet a swtich, when hop_limit = 0 the packet will be drop.
Parser and deparser no need change sbecause extract and emit work on the whole header. The host need to change which in send.py and receive.py it need to add field for hop_limit.

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->

In tunnel.p4, I added an 8-bit hop_limit at the end of myTunnel_t, decrement it in myTunnel_forward, and drop the packet at the start of ingress if hop_limit is 0. The parser and deparser are unchanged because extract and emit handle the whole header.

In send.py, the MyTunnel class has a new ByteField for hop_limit (default 8), plus an optional --hops argument for testing the drop. In receive.py, the same field was added so the 5-byte header is decoded correctly, and it now prints hop_limit.

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

Same loop but this time the packet number doesnt increase after 5 seconds, meaning it is drop which match my design.

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

I predict that s1 will handle the tunnel header correctly, but s2 will misread it, because s2 expects a 4-byte header while the hosts now send 5 bytes. s2 will read the hop_limit byte as the first byte of the IPv4 header, so every IPv4 field is shifted by one byte, and its checksum update will overwrite the real protocol byte. I expect tunnelled packets to still be forwarded by dst_id, but receive.py will not be able to decode them, so h2 and h3 will print nothing for them, while plain IPv4 will still arrive. I expect the loop to still end, but later than before: only s1 decrements the hop limit, so s1 should see 9 passes and s2 should see 8, about 17 in total instead of 9.

Assigns a new EtherType (e.g., 0x1213) to the updated header while retaining parser support for the legacy type (0x1212) should able to perform the rollout

Data-plane loop protection (the 8-bit hop_limit) catches real-time transient loops and in-flight packet cycling caused by asynchronous table updates, link-failure race conditions, or corrupted runtime forwarding state that static analysis cannot foresee. Conversely, control-plane loop protection (such as check_paths()) performs global graph topology analysis before deploying rules, catching structural routing loops and configuration errors before any packet is transmitted, thereby preventing the line-rate bandwidth consumption and packet drops that occur when relying solely on data-plane drop logic.

Adding an 8-bit hop_limit field increases header payload overhead by 1 byte, slightly expands the parser bus allocation, and consumes one 8-bit ALU operation within an Ingress Match-Action stage to perform the decrement and zero-comparison. Because hardware switch pipelines execute instructions deterministically in hardware stages, this field processing incurs zero throughput degradation and runs at full line rate, consuming only a small slice of your stage ALU resources and parser memory budget.

## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

I let claude explain how the code basically work to me
I draft the hop limit and let claude to refine and complete it
For the questions i draft the answer and let claude to refine them also 

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
