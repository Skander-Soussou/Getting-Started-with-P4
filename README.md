- [Getting started with P4](#getting-started-with-p4)
  * [P4 Tutorial](#p4-tutorial)
    + [P4 Tutorial Installation](#p4-tutorial-installation)
    + [Start the tutorial](#start-the-tutorial)
    + [What if stuck](#what-if-stuck)
  * [P4 Cheat Sheet](#p4-cheat-sheet)
    + [core.P4](#corep4)
      - [error types](#error-types)
      - [functions used for packetsin used in parser](#functions-used-for-packetsin-used-in-parser)
      - [functions used for packetsout used in deparser](#functions-used-for-packetsout-used-in-deparser)
      - [actions](#actions)
      - [match-kind](#match-kind)
    + [v1model.p4](#v1modelp4)
      - [types of metadata](#types-of-metadata)
      - [P4 program structure](#p4-program-structure)
  * [Basic example](#basic-example)
    + [Json topology](#json-topology)
    + [P4 code](#p4-code)
    + [The static control plane for switch number 1](#the-static-control-plane-for-switch-number-1)

# Getting started with P4
## P4 Tutorial
### P4 Tutorial Installation
You can check the [P4 installation steps](/installation.md)
### Start the tutorial
Now that you're done with the basis of P4 language and the installation you can start the [tutorial](https://github.com/p4lang/tutorials)
### What if stuck
Here you can find [useful links](/Helpful_links.md) that helped me as a total beginner in networking and P4 language understand.
## P4 Cheat Sheet
### core.P4
Every P4(16) code starts with #include <core.p4> defined at https://github.com/p4lang/p4c/blob/master/p4include/core.p4
In core.p4 we have the main language features lik
#### error types
    NoError
    PacketTooShort
    NoMatch
    StackOutOfBounds
    HeaderTooShort
    ParserTimeout
    ParserInvalidArgument
#### functions used for packetsin used in parser
    extract (read bytes and advance cursor)
    advance (advance cursor)
    lookahead (read bytes without advancing the cursor)
#### functions used for packetsout used in deparser
    emit (writing a header in the output packet)
#### actions
    NoAction()
#### match-kind
    exact (matches if both arguments are exactly the same)
    lpm   (matches with the longest prefix)
    ternary (matches if both adresses have same mask)
### v1model.p4    
   We're normaly also gonna include v1model.p4 with #include <v1model.p4> 
   defined at https://github.com/p4lang/p4c/blob/master/p4include/v1model.p4
   
   v1model.p4 is the most used P4_16 architecture
#### types of metadata
    standard_metadata:
    * ingress_port  -> the port that the data came from
    * egress_spec   -> we can assign the  output-port for the packet
    * egress_port   -> the output-port (only accessible in the egress processing)
    * packet_lenght
    ...
    
    intrinsic metadata: are user defined metadata
    struct{
       bit<9>      egress_spec;  // you can youse this to hold a copy for example
    }
 #### P4 program structure
 | Section                  | example       |
| ----------------------- |:---------------:|
| Headers definition      | header ethernet |
| Parser                  | parser MyParser(packet_in packet,out headers hdr,inout metadata meta,inout standard_metadata_t standard_metadata) |
| checksum verification   | control MyVerifyChecksum(inout headers hdr, inout metadata meta) |
| Ingress processing      | control MyIngress(inout headers hdr,inout metadata meta,inout standard_metadata_t standard_metadata) |
| Egress processing       | control MyEgress(inout headers hdr,inout metadata meta,inout standard_metadata_t standard_metadata) |
| checksum computation    | control MyComputeChecksum(inout headers  hdr, inout metadata meta) |
| Deparser                | control MyDeparser(packet_out packet, in headers hdr) |
| Switch (main)           |V1Switch(MyParser(),MyVerifyChecksum(),MyIngress(),MyEgress(),MyComputeChecksum(),MyDeparser()) main; |

## Basic example
### Json topology
![](images/pod-topo.png)
Here's the following Json topology code:
``` json
{
    "hosts": {
        "h1": {"ip": "10.0.1.1/24", "mac": "08:00:00:00:01:11",
               "commands":["route add default gw 10.0.1.10 dev eth0",
                           "arp -i eth0 -s 10.0.1.10 08:00:00:00:01:00"]},
        "h2": {"ip": "10.0.2.2/24", "mac": "08:00:00:00:02:22",
               "commands":["route add default gw 10.0.2.20 dev eth0",
                           "arp -i eth0 -s 10.0.2.20 08:00:00:00:02:00"]},
        "h3": {"ip": "10.0.3.3/24", "mac": "08:00:00:00:03:33",
               "commands":["route add default gw 10.0.3.30 dev eth0",
                           "arp -i eth0 -s 10.0.3.30 08:00:00:00:03:00"]},
        "h4": {"ip": "10.0.4.4/24", "mac": "08:00:00:00:04:44",
               "commands":["route add default gw 10.0.4.40 dev eth0",
                           "arp -i eth0 -s 10.0.4.40 08:00:00:00:04:00"]}
    },
    "switches": {
        "s1": { "runtime_json" : "pod-topo/s1-runtime.json" },
        "s2": { "runtime_json" : "pod-topo/s2-runtime.json" },
        "s3": { "runtime_json" : "pod-topo/s3-runtime.json" },
        "s4": { "runtime_json" : "pod-topo/s4-runtime.json" }
    },
    "links": [
        ["h1", "s1-p1"], ["h2", "s1-p2"], ["s1-p3", "s3-p1"], ["s1-p4", "s4-p2"],
        ["h3", "s2-p1"], ["h4", "s2-p2"], ["s2-p3", "s4-p1"], ["s2-p4", "s3-p2"]
    ]
}
```
### P4 code
``` P4
/* -*- P4_16 -*- */
#include <core.p4>
#include <v1model.p4>

const bit<16> TYPE_IPV4 = 0x800; 

/*************************************************************************
*********************** H E A D E R S  ***********************************
*************************************************************************/

typedef bit<9>  egressSpec_t; /* egressSpec_t now correspond to bit<9> (unsigned integer 9 bits width) */
typedef bit<48> macAddr_t;
typedef bit<32> ip4Addr_t;

header ethernet_t { 
    macAddr_t dstAddr; /* correspond to bit<48> dstAddr */
    macAddr_t srcAddr;
    bit<16>   etherType;
}

header ipv4_t {
    bit<4>    version;
    bit<4>    ihl;
    bit<8>    diffserv;
    bit<16>   totalLen;
    bit<16>   identification;
    bit<3>    flags;
    bit<13>   fragOffset;
    bit<8>    ttl;
    bit<8>    protocol;
    bit<16>   hdrChecksum;
    ip4Addr_t srcAddr;
    ip4Addr_t dstAddr;
}

struct metadata { /* we don't need intrinsic metadata in this basic example */
    /* empty */
}

struct headers { /* we need to have a structure headers that regroups all the headers we're using*/ 
    ethernet_t   ethernet;
    ipv4_t       ipv4;
}

/*************************************************************************
*********************** P A R S E R  ***********************************
*************************************************************************/

parser MyParser(packet_in packet, /* simply the packet in */
                out headers hdr,  /* an empty headers struct defined earlier */
                inout metadata meta, /* intrinsinc metadata not defined in this example */
                inout standard_metadata_t standard_metadata) /* standard metadata defined in the v1model */ { 
                
     /* we can't modify the parameters it's hard-coded in the v1model */
    
    /* consider the parser as a state machine */
    state start { /* the states start and accept are hard-coded in the v1model */
    /* there's also the reject state if needed */
        transition parse_ethernet; /* we transition to parse_ethernet state */
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) { /* transition select is like a switch case */
            TYPE_IPV4: parse_ipv4; /* if etherType = TYPE_IP4 we transition to parse_ipv4 */
            default: accept; /* else we transition to accept state */
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }

}

/*************************************************************************
************   C H E C K S U M    V E R I F I C A T I O N   *************
*************************************************************************/

/* MyVerifyChecksum is a control block that performs a checksum on a parsed packet */
/* in this block you can modify metadata fields (for example setting error-flags) */

control MyVerifyChecksum(inout headers hdr, inout metadata meta) {   
    apply {  }
}


/*************************************************************************
**************  I N G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t standard_metadata) {
    action drop() {
        mark_to_drop(standard_metadata); /* we define and action in order to drop the packet */
    }
    
    /* the action's parameters are initialized by the control plane */
    action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port; /* we change on which port the packet is gonna be sent */
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr; /* we initialise the new source mac adresse to the old mac destination adresse */
        hdr.ethernet.dstAddr = dstAddr; /* initialise the new destination mac adresse to the action's parameter */
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1; /* decrease the ipv4 time to live */
    }
    
    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm; /* matches if the ipv4 destination adress has the longest prefix */ 
        }
        actions = { /* list of all the actions possible if it matches*/
            ipv4_forward;
            drop;
            NoAction;
	    /* the control plane chooses the action for each match */
        }
        size = 1024; /* how many entries should the table alocate */
        default_action = drop(); /* default action can be changed by the control plane (unless you declare it as const) */
    }
    
    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        }
    }
}

/*************************************************************************
****************  E G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyEgress(inout headers hdr,
                 inout metadata meta,
                 inout standard_metadata_t standard_metadata) {
    apply {  }
}

/*************************************************************************
*************   C H E C K S U M    C O M P U T A T I O N   **************
*************************************************************************/

/* calculate checksum of the outgoing packet before the parser */

control MyComputeChecksum(inout headers  hdr, inout metadata meta) {
     apply {
	update_checksum(
	    hdr.ipv4.isValid(),
            { hdr.ipv4.version,
	      hdr.ipv4.ihl,
              hdr.ipv4.diffserv,
              hdr.ipv4.totalLen,
              hdr.ipv4.identification,
              hdr.ipv4.flags,
              hdr.ipv4.fragOffset,
              hdr.ipv4.ttl,
              hdr.ipv4.protocol,
              hdr.ipv4.srcAddr,
              hdr.ipv4.dstAddr },
            hdr.ipv4.hdrChecksum,
            HashAlgorithm.csum16);
    }
}

/*************************************************************************
***********************  D E P A R S E R  *******************************
*************************************************************************/

control MyDeparser(packet_out packet, in headers hdr) {
    apply { 
    	/* emit the headers followed by the data */
        packet.emit(hdr.ethernet); 
        packet.emit(hdr.ipv4);
    }
}

/*************************************************************************
***********************  S W I T C H  *******************************
*************************************************************************/

V1Switch(
MyParser(),
MyVerifyChecksum(),
MyIngress(),
MyEgress(),
MyComputeChecksum(),
MyDeparser()
) main;
```
### The static control plane for switch number 1
As said earlier the control plane defines the default action, the action for every match and the action parameters for tables.
``` json
{
  "target": "bmv2",
  "p4info": "build/basic.p4.p4info.txt",
  "bmv2_json": "build/basic.json",
  "table_entries": [
    {
      "table": "MyIngress.ipv4_lpm",
      "default_action": true,
      "action_name": "MyIngress.drop",
      "action_params": { }
    },
    {
      "table": "MyIngress.ipv4_lpm",
      "match": {
        "hdr.ipv4.dstAddr": ["10.0.1.1", 32]
      },
      "action_name": "MyIngress.ipv4_forward",
      "action_params": {
        "dstAddr": "08:00:00:00:01:11",
        "port": 1
      }
    },
    {
      "table": "MyIngress.ipv4_lpm",
      "match": {
        "hdr.ipv4.dstAddr": ["10.0.2.2", 32]
      },
      "action_name": "MyIngress.ipv4_forward",
      "action_params": {
        "dstAddr": "08:00:00:00:02:22",
        "port": 2
      }
    },
    {
      "table": "MyIngress.ipv4_lpm",
      "match": {
        "hdr.ipv4.dstAddr": ["10.0.3.3", 32]
      },
      "action_name": "MyIngress.ipv4_forward",
      "action_params": {
        "dstAddr": "08:00:00:00:03:00",
        "port": 3
      }
    },
    {
      "table": "MyIngress.ipv4_lpm",
      "match": {
        "hdr.ipv4.dstAddr": ["10.0.4.4", 32]
      },
      "action_name": "MyIngress.ipv4_forward",
      "action_params": {
        "dstAddr": "08:00:00:00:04:00",
        "port": 4
      }
    }
  ]
}
```
