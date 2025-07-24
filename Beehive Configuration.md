Beehive configuration is driven by an XML file associated with each particular design instantiation. Typically these XML files are in the directory for the design we are building, since it defines a particular topology, but theoretically they could be anywhere. This configuration is used mainly to generate certain portions of code, but is also used for message deadlock analysis since much of the information needed can be derived from this XML configuration file.
# Tile Generation
Beehive generates certain files that have to do with the tile topology and logical routing using a Python preprocessor that allows Python prints embedded in the file to output Verilog. In particular, the files Beehive currently generates are the file that instantiates and wires up all the tiles and the CAMs that control message routing in the RX path.

We choose to do actual code generation instead of using (System)Verilog `generates` and `for` loops, because of the usefulness of being able to actually look at the generated wires for debugging both the instantiation of the design itself and during waveform analysis.

The generator code and preprocessor code lives in `tile_generator`.
## Overall Design Goal
The goal of the code generation is to automate some of the very structured, but tedious code that exists when building the design, such as connecting wires between tiles. This is especially true as we move to supporting more than one physical NoC. Notably, the goal of the code generation is NOT to replace Verilog or to define some higher level language.

This is a revamp of the original system where we did try and generate the whole tile, but there are a couple cases where tiles have unique interfaces that start to introduce complexities into the code generation that started to bring it dangerously close to us developing a small language.

In particular, while all tiles will have the same set of NoC ports, not all tiles will have the same set of ports in general. For example, the Ethernet tiles have additional ports to connect to the MAC and memory tiles have additional ports for connecting to a memory interface. There are also certain endpoints that are wrapped in one module at the level of instantiating tiles. For example, the TCP has one endpoint for send and one for receive. However, the TCP engine receive and transmit side need to share a fair bit of state, so it is easier if they are wrapped in one module
# Format
The XML is parsed to turn it into a Python configuration object that can be used when generating code. An example XML is in `cocotb_testing/udp_echo/tile_config.xml`

The top level XML tag is `design`. Within `design` there are two main subtags: `nocs` which controls the NoC setup for the design and `tiles` which makes up the bulk of the file and specifies the tiles and their topology.
## NoCs
This tag controls the number of NoCs generated. Each NoC is specified with the tag `noc`. The subelements for a `noc` are:
- `noc_name`: a string that is used when generating
- `noc_width_def`: a string that provides the name of the `define` macro that sets the data width for the NoC. This should match the `define` in the Verilog as it’s used when generating the data wires for the NoC.

## Tiles
Since we are using a 2D mesh, there should be two subelements to specify the mesh dimensions:
- `num_x_tiles` : number of tiles in the x-dimension
- `num_y_tiles` : number of tiles in the y-dimension
(**TODO: if we ever enable more topologies, we should change these**)
These subelements allow us to automatically emit empty tiles that just contain NoC routers to make things rectangular.

Then there are some number of `endpoint` elements. An endpoint represents a NoC router with unique (X, Y) coordinates (given by the `DST_X` and `DST_Y` fields within the NoC header flit). Each `endpoint` will contain one or more `interface` elements. These represent the actual destination for a NoC message within a tile (given by the `DST_FBITS` field within the NoC header flit.
### Endpoint
For every endpoint, the Python code can generate the “standard” NoC ports with the appropriate wire connections

The required subelements are:
- `endpoint_name`: a string that we can use to reference this specific endpoint. This should be unique across the entire design. It becomes the key for the Endpoint object in the dictionary under the top-level configuration object. This should probably be the instance name that you’re planning to instantiate the module with
- `port_name`: the name for the tile ports
- `endpoint_x`: the x-coordinate for this endpoint
- `endpoint_y`: the y-coordinate for this endpoin
- `interfaces` which specifies in more detail what is addressable from the NoC
### Interface
An object within an endpoint that is accessible over the NoC. The required subelements are:
- `if_name`: a name unique within the endpoint to specify it. Generation code may look for an interface using this name, so make it useful
- `fbits`: at what fbits is this interface reachable.
- `parent_endpoint`: just a pointer back to the endpoint that the interface is contained in
An optional subtag that is used is `dsts`. This should contain some number of subelements that have the tag `dst_endpoint`. These are used to generate the CAMs for routing as well deadlock analysis.
#### dst_endpoint:
These define another endpoint that an interface will send to. These are just turned into dictionaries. These dictionaries should contain all the information that their interface parent requires to send to them. The required subelements are
- `endpoint_name`: this is a string that specifies the destination endpoint that this interface is sending to. Note that this name needs to be exactly the name of another endpoint that exists in the design
- `if_name`: this is a string that identifies the interface in the endpoint that the interface is sending to. This name needs to exactly be the name of an interface within `endpoint_name`
- `noc_name`: which NoC this request will be carried on. This is only used for deadlock analysis and not for code generation at all
##### CAM generation tags
If the endpoint is something that needs to be known to generate a CAM, it should have at least the following tags:
- `cam_target`: this is an empty/self-closing tag that just specifies this should go in the cam
- `endpoint_tag`: what Verilog expression should be matched on
The X and Y coordinates of the destination are already available by using the `endpoint_name` specified to look up the Endpoint object in the top level configuration dictionary. However, interface fbits can also be specified here and such
### Deadlock analysis tags
- `noc_name`: which NoC this interface is sending requests on
- `depends_on`: for interface-interface dependencies within a single endpoint (never goes out onto the NoC). Loopback dependencies (which will use a router) should use the normal `dst_endpoint` method
# Using the Configuration Object for Code Generation
It’s probably easier to look at an example of this. One example is in `cocotb_testing/udp_echo/udp_echo_top_gen.sv.pyv`. Here’s some brief detail of how the generation is expected to be used.

Basically, the top-level module ports are defined. Then somewhere, the correct modules are imported and the tile config file is read to construct the configuration object. The configuration object has a method that can be used to generate declarations for all the wires that are needed to connect tiles together.

Then the programmer will go thru and instantiate all the tiles, except they don’t have to write out all the ports. In particular, they can use the dictionary in the configuration object to look up an endpoint by `endpoint_name`, which is probably the instantiated module’s name, and then they can use the endpoint object to generate the standard NoC ports with the appropriate wires.

This means that for tiles like IP or UDP, which only use the standard NoC ports, the Endpoint object is able to generate all the ports, but the programmer will write some one-off ports for the Ethernet tiles which use unique connections
# Using the Configuration Object for deadlock analysis
- This process is done by `util/scripts/deadlock_check.py`, but documented here at a high level for posterity
- Endpoints are turned into two nodes: one for input, one for output
- Each endpoint also generates nodes representing its outgoing links. Incoming links are owned by the endpoints for which they are outputs
	- Extra NoCs are represented by generating additional nodes
- Interfaces rely on the output node to send and are depended on by the input endpoint node to receive
- Edges should be added starting at an interface and then traversing the nodes that correspond to the NoC links to reach the destination endpoint.
- The graph can then be checked for cycles
	- Note that this is a conservative, static analysis. It may be possible to reason that a deadlock doesn't occur in practice due to message lengths