# PA1 Capture Analysis

_Your capture, `evidence/pa1-capture.pcap`, holds two ETP sessions from your own
programs: a PUT, then a GET of the same file. This analysis is about the **GET
session** unless a question says otherwise. Use Wireshark (or tshark) to read the
capture. The numbers must come from your capture: they are checked against it._

## How I captured it

_Paste the exact commands you ran (capture, server, client, ss), and the size of
the file you transferred._

```
Server: make run-server PORT=5000 ROOT=server-files



Capture: sudo tcpdump -i lo -nn -s 200 -U -w - 'tcp port 5000' > evidence/pa1-capture.pcap

File creation:
head -c 300000 /dev/urandom > /tmp/etp-pa1-upload.bin

PUT and GET client commands
PUT:
./etp-client --host 127.0.0.1 --port 5000 --name kaelyn put /tmp/etp-pa1-upload.bin test-upload.bin
GET:
./etp-client --host 127.0.0.1 --port 5000 --name kaelyn get test-upload.bin /tmp/etp-pa1-download.bin


TIME-WAIT command using ss
TIME-WAIT:
./etp-client --host 127.0.0.1 --port 5000 --name kaelyn list > /dev/null
ss -tan state time-wait '( sport = :5000 or dport = :5000 )' | tee evidence/time-wait.txt


Transferred file size: 300,000 bytes
```

## Facts

_Fill in every value from **your** capture, for the **GET** session. Integers in
decimal. Sequence numbers are the **absolute** (raw) values — in Wireshark, turn
off "relative sequence numbers" (Edit > Preferences > Protocols > TCP) or read
the "Sequence Number (raw)" field. With tshark, add
`-o tcp.relative_sequence_numbers:FALSE`. Do not change the key names or the
`etp-facts` label: a script reads this block._

```etp-facts
server_port: 5000
client_port: 54206
client_isn: 3941957868
server_isn: 4044087922
client_mss: 65495
server_to_client_payload_bytes: 300038
server_to_client_data_segments: 12
first_fin_from: server
```

## Q1. The handshake

_Give the frame number, flags, and raw sequence and acknowledgment numbers of the
three handshake segments of the GET session. Explain why the acknowledgment number
in the third segment is what it is. What MSS did the client advertise, and what on
your machine determines that value?_

Frame 29: SYN, SEQ 3941957868, ACK N/A  
Frame 30: SYN-ACK, SEQ 4044087922, ACK 3941957869  
Frame 31: ACK, SEQ 3941957869, ACK 4044087923

The client sent SYN, the server replied with SYN-ACK, and the client completed the handshake with ACK. The final ACK is 4044087923 because SYN consumes one sequence number (4044087922 + 1).

The client advertised an MSS of 65495 bytes because Linux's loopback interface has a larger MTU than standard Ethernet.

## Q2. Your protocol on the wire

_Find the segment that carries your client's GET request. Give its frame number,
paste the first 8 payload bytes in hex, and decode each header field (VERSION,
TYPE, STATUS, RESERVED, LENGTH). Do the same for the header of the server's
GET_OK. Does GET_OK's LENGTH equal the size of your file? Show the arithmetic that
connects `server_to_client_payload_bytes` to the three responses the server sent in
this session._

GET request (Client → Server)
- Frame: 36
- First 8 bytes (hex): 01 03 00 00 00 00 00 0F
- VERSION: 01 — ETP version 1
- TYPE: 03 — GET
- STATUS: 00 — No error
- RESERVED: 00 — Zero
- LENGTH: 0000000F — 15 bytes (filename test-upload.bin)
GET_OK response (Server → Client)
- Frame: 37
- First 8 bytes (hex): 01 83 00 00 00 04 93 E0
- VERSION: 01 — ETP version 1
- TYPE: 83 — GET_OK
- STATUS: 00 — Success
- RESERVED: 00 — Zero
- LENGTH: 000493E0 — 300,000 bytes
Does GET_OK's LENGTH match the file size? Yes. Both are 300,000 bytes.

Server-to-client TCP payload calculation:

HELLO_OK = 8-byte header + 14-byte payload = 22 bytes
GET_OK   = 8-byte header + 300,000-byte file = 300,008 bytes
BYE_OK   = 8-byte header + 0-byte payload = 8 bytes

Total = 22 + 300,008 + 8 = 300,038 bytes

This matches the server_to_client_payload_bytes
value of 300,038 recorded in the packet capture.

## Q3. A byte stream, not messages

_How many segments carried the **GET_OK response** (not counting the segments
that carried HELLO_OK and BYE_OK), and what were their payload sizes? Were they
all the same size? Were they all the MSS? Did any one segment carry the
end of one ETP message and the start of the next, or did one ETP message span
several segments? Explain what this means for code that reads an ETP message, and
point to the place in your own code that deals with it.

TCP segments carrying GET_OK	10
Frames	37, 38, 40, 42, 43, 45, 46, 48, 49, 51
Segment sizes	Nine × 32,768 bytes; one × 5,096 bytes
Total GET_OK bytes	300,008
Client-advertised MSS	65,495 bytes
Were all segments equal?	No
Were all segments equal to the MSS?	No
Did an ETP message span several segments?	Yes
Did a segment combine two ETP messages?	Not in this capture

TCP is a byte-stream protocol. It does not preserve application-message boundaries
The application cannot assume that one TCP read returns an entire ETP message.

Our implementation handles this in src/transport_tcp.py using recv_exact(), which repeatedly calls recv() until the requested number of bytes has been received. In src/etp.py, read_message() uses this function to receive the ETP header and payload.

In this capture, one GET_OK message spans 10 TCP segments. No observed segment contains the end of one ETP message and the beginning of another.

## Q4. Teardown and TIME-WAIT

The server sent the first FIN in frame 54, followed by the client's FIN in frame 55 and the server's final ACK in frame 56. This follows the ETP/1 specification, which requires the server to close after sending BYE_OK. The server (`127.0.0.1:5000`) entered TIME-WAIT because it initiated the shutdown. Linux typically maintains TIME-WAIT for about 60 seconds. SO_REUSEADDR allows the server to restart on the same port while previous connections remain in TIME-WAIT.

TIME-WAIT evidence from `evidence/time-wait.txt`:

```text
0  0  127.0.0.1:5000  127.0.0.1:46534
```

This TIME-WAIT evidence was collected during a separate LIST session, while the packet capture analysis focuses on GET.




## Q5. What an observer learns

_Suppose someone else obtained this capture file. List specifically what they
learn about your session (names, sizes, timing, anything else). You captured with a
200-byte snap length: what does that hide, and what does it not hide? Which kind of
protection would stop a real on-path observer from learning these things, and what
would it still not hide?_

An observer with this capture could identify the client name (`kaelyn`), filename (`test-upload.bin`), file size (300,000 bytes), IP addresses, ports, and transfer timing. The 200-byte snap length limits how much file data is recorded but does not hide protocol headers, filenames, or metadata. TLS encryption would protect the filenames, messages, and file contents from an on-path observer, but IP addresses, TCP ports, packet timing, and approximate transfer sizes would remain visible.