### OSI 7-Layer Model

- Media layers
  - Layer 1: Physical
    - transmission and reception of raw bit streams between a device and a shared physical medium
    - voltage levels, timing, rates, distances, modulations, connectors
    - Hub (layer 1 device) only supports broadcasting (one to everyone)
    - only supports one transmitter at a time
    - no media access control
    - no collision detection
    - no uniquely identified devices
    - no device -> device communication
  - Layer 2: Data Link
    - MAC addres
    - frames
      - preamble: 56 bits
      - start frame delimiter: 8 bits
      - destination MAC address: 48 bits
      - source MAC address: 48 bits
      - ether type: 16 bits (example: IP)
      - payload: 46 - 1500 bytes
      - frame check sequence: 32 bits
    - checks for carrier signal before sending (collision avoidance)
    - CSMA/CD
      - CD - collision detection
        - jam signal
        - random back off
    - Switch (layer 2 device)
      - understand frames and MAC addresses
      - maintain a MAC address table (MAC address -> switch port)
  - Layer 3: Network
    - able to connect several Layer 2 networks
    - adds cross-network IP addresses and routing
    - Router (layer 3 device) is able to unwrap and re-wrap frames
    - uses **packets** (IP-packets) as a data transfer unit
    - IP packet (among other fields) contains:
      - source IP address
      - destination IP address
      - protocol (Level 4, ICMP/TCP/UDP)
      - TTL (hop limit)
    - IP v4
      - dotted-decimal notation
        - human-readable format
        - 4 numbers 0-255, separated by dots
        - two "virtual" parts:
          - "network" part
          - "host" part
      - 32 bits in total (four 8-bits octets, read from left to right)
      - if the "network" components of two IP addresses are the same, the devices are local
      - IP addresses might be assigned:
        - statically: by humans
        - automatically: by DHCP (Dynamic Host Configuration Protocol) service
    - default gateway: IP address where packets are forwarded to if destination is not local
    - subnet mask
      - allows an IP device to determine whether IP address is local (on the same network) or not
      - examples:
        - `255.255.0.0` (same as `/16` prefix)
      - in binary:
        - `1` represents the network part
        - `0` represents the host part
        - starting address of the local network have all `0` in the host part
        - ending address of the local network have all `1` in the host part
      - larger prefix -> more specific route
        - `0.0.0.0/0` - most generic (all IP addresses)
          - in route table it is called the default route
        - `52.217.13.0/24` - pretty specific (256 - 1 IP addresses)
        - `52.217.13.27/32` - single IP address
    - route table
      - destination IP mask -> next hop / target
      - if there are multiple matching destination masks, more specific is used
      - might be populated statically or dynamically via the border gateway protocol (BGP)
      - BGP allows routers to communicate exchanging routing information
    - Address Resolution Protocol (ARP)
      - used when there is a need to find the MAC address for a given IP address to:
        - take a level 3 packet
        - incapsulate it into a frame
        - send the frame to a MAC address
      - runs between layer 2 and layer 3
      - ARP algorithm in local network:
        - ARP broadcasts to all MAC addresses (all Fs) asking for a MAC address of a specific IP address
        - ARP on the side of a specified IP address responds with a MAC address
        - ARP encapsulates the IP packet into the L2 frame using the obtained MAC address as a destination
        - the frame transmitted via the L1
      - Routing algorithm across several networks works by wrapping/unwrapping the same IP packet multiple times in different frames, each time using MAC address of a next hop (next router or final destination)
    - IP packets sent as independent entites, so:
      - they may arrive out-of-order
      - it's hard to "group" them on an application level
    - IP packets can go missing
    - Level 3 doesn't have flow control -> destination oversaturation issue
- Host layers
  - Layer 4: Transport
    - adds two protocols:
      - TCP - Transmission Control Protocol
        - slower
        - reliable
        - connection-oriented
      - UDP - User Datagram Protocol
        - faster
        - less reliable
    - TCP/IP means TCP running on top of IP
    - TCP introduces segments, which are placed inside the IP packets
    - TCP segments contains:
      - TCP header:
        - source and destination ports for routing
        - sequence number for ordering
        - acknowledgement field for indicating that segments up to X were received
        - flags'n'things
          - FIN - to close the connection
          - ACK - for acknowledgement
          - SYN - to synchronize sequence numbers
        - TCP window: number of bytes receiver is ready to receive before ack (flow control)
        - checksum (for error checking)
        - urgent pointer (?)
        - options
        - padding
    - TCP establishes a connection
      - bidirectional
      - reliable
      - using a specific server port and a random client port
    - ports can be:
      - well known (80, 443, etc; usually on the server)
      - ephemeral (temporary random port; usually on the client)
    - 3-way handshake
      - client sends a segment to the server with SYN set to `cs` ISN (initial sequence number)
      - server reponds with a segment containing:
        - SYN: `ss` ISN
        - ACK: `cs + 1`
      - client reponds with a segment containing:
        - SYN: `cs + 1`
        - ACK: `ss + 1`
    - stateless firewall requires both rules (inbound and outbound)
    - stateful firewall requires only one rule
  - Layer 5: Session
  - Layer 6: Presentation
  - Layer 7: Application

Layer X device has layers 1-X capabilities.

Layer X <-> Layer X communication abstracts all underlying levels.
