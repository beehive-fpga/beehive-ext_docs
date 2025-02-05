Each layer passes a NoC message to the next layer. A NoC message consists of one or more flits, which for Beehive is just the width of the NoC. The general format is a flit (a header flit), which contains mostly information that is relevant to the NoC, followed by a flit (a metadata flit), which carries information specifically for the protocol, and then some number of flits that contains the payload of the packet.

## Header flit

The struct representing the NoC header flit is in `network_tiles/tcp_hw/include/noc_struct_pkg.sv`. It’s the `noc_hdr_flit_struct`. For historical reasons, there’s `noc_header_flit_{1-3}`. Ignore them

The generally relevant fields are:

- `dst_x`: this is the x-coordinate of the tile the message is being sent to
- `dst_y`: the y-coordinate of the tile the message is being sent to
- `dst_fbits`: this is used for directing the message to the appropriate module within the tile
- `msg_len`: this is the number of flits in the message following the header flit, so if the message just consists of the header flit, this field will be 0
- `msg_type`: a constant that just represents what the message is
- `metadata_flits`: the number of metadata flits following this message. This should be ≤ `msg_len`
- `src_x`: this is the x-coordinate of the tile sending the message
- `src_y`: this is the y-coordinate of the tile sending the message
- `src_fbits`: this is used to identify the module within the tile the message was sent from

Everything else can be set to 0

## Metadata flit

This is a protocol specific flit. The structs for these are found in `include/beehive_{protocol}_msg.sv`, so the flits for the UDP messages are in `include/beehive_udp_msg.sv`. Generally the fields are fairly straightforward in here

## Payload flits

Just the data payload broken into flits sized chunks