# PA1 Design — ETP/1 Client and Server

_One to two pages. Short, specific answers beat long general ones. Every section
should be true of the code you submit. Delete these italic prompts when you write._

## 1. Architecture

Client.py connects to the server and sends a HELLO. Then, depending on the arguments for run(), it gives a LIST, GET, or PUT command. When it is done, it sends BYE and disconnects.

Server.py creates a listens for connections and creates a thread for each one in main(). After the connection is established, it goes to handle_sessions(), which is a loop where it checks for which request it got and performs that task.

Etp.py contains all the constants like the message and error types and MAX_LEN. It also builds an ETP message with its header and payload in encode(), and read_message() uses recv_exact to read a message and return the tuple of (version, mtype, status, length, payload).

Transport_tcp.py is the TCP protocol. It is the only file to use socket and the six functions of listen, accept, connect, send_all, recv_exact, and close

Transport.py is the file that client, server, and etp use to access the transfer protocol. It imports all from transport_tcp.py

## 2. The transport seam

The transport is in transport_tcp.py while transport.py imports it for other files to use. Listen creates and returns the socket while using SO_REUSEADDR. Accept blocks for a connection and returns the connection and address. Connect creates a socket and connects using host and port. Send_all sends every byte of data given to it. Recv_exact loops sending chunks of data until n bytes are sent; if the connection closes half-way through, it sends ConnectionClosed(clean=False). Close half-closes with shutdown(socket.SHUT_WR) and then closes the socket.

When we change from TCP to UDP, we will have to write a new transport to replace transport_tcp.py. Then, transport.py will change to import from that file instead. All other files should be the same.

## 3. Framing

The codes reads ETP messages using recv_exact. Because recv_exact loops until the 8 bytes of the header arrive, it can deal with a header that arrives in peaces or multiple messages. VERSION and LENGTH are checked in read_message() in etp.py and handle_session() in server.py. We need to check these values before reading the payload as they can desynchronize the connection if the version is wrong or the message is too long.

## 4. Concurrency

The server uses threads to seperate each session, which makes so one client cannot stall the whole program. The only true shared data are the files in the root directory. Lock is used when printing logs to make sure it isn't combined with other sessions, and the PUT creates a temporary file, using the thread_id to ensure a unique name, so that GET can't see it while it is being written.

## 5. Errors and disconnects

Fatal errors, such as BAD_VERSION, TOO-LARGE, and NOT_READY, log a response and then instantly closes the connection. Non-fatal errors get an error response but still continues the session loop, allowing for more requests. Disconnecting in the middle of a GET raises an OSError, which is handled by a try except block. Afterwards, the connection is closed on the server. The client has 4 different exit codes: 0, 1, 2, and 3. 0 is used when the client exits normally with BYE_OK. 1 is used whenever it gets an error from the server. 2 is used whenever there is an error unrelated to the server, such bad arguments. 3 is used whenever the server's digest from the PUT_OK is different from the local digest.

## 6. Threat model

Anyone who can see the traffic from the server can attack it. One easy attack that the server does not resist is denial of service, as there is no cap on connections or any automatic closing of idle sessions. The server does resist path traversal in the file system because the file name validation is strict and does not allow for names to point outside of the root directory.

## 7. Testing

I mainly tested by doing the etp_selfcheck.py and asking Claude to try to find bugs in the code. One bug I found using Claude is that the ready variable used in the server to make sure that HELLO was sent first was in the main loop of handle_session(). This causes the HELLO to not save and other requests to get the NOT_READY error.
