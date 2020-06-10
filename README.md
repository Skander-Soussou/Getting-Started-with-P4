# P4 Tutorial

[Va ici](./test)

## core.P4
Every P4(16) code starts with #include <core.p4> defined at https://github.com/p4lang/p4c/blob/master/p4include/core.p4
In core.p4 we have the main language features lik
### error types:
    NoError
    PacketTooShort
    NoMatch
    StackOutOfBounds
    HeaderTooShort
    ParserTimeout
    ParserInvalidArgument
### functions used for packets_in (parser):
    extract
    advance
    lookahead
### functions used for packets-out (deparser):
    emit
### actions:
    NoAction()
### match-kind:
    exact (matches if both arguments are exactly the same)
    lpm   (matches with the longest prefix)
    ternary (matches if both adresses have same mask)
## v1model.p4    
    We're normaly also gonna include v1model.p4 with #include <v1model.p4> 
    defined at https://github.com/p4lang/p4c/blob/master/p4include/v1model.p4 
    v1model.p4 is the most used P4_16 architecture
### types of metadata:
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
 ### P4 program structure:
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

## Basic example :
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
### P4 code :
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
        transition parse_ethernet; /* we transition to parse_ethernet state */
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            TYPE_IPV4: parse_ipv4;
            default: accept;
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
        mark_to_drop(standard_metadata);
    }
    
    action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }
    
    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            ipv4_forward;
            drop;
            NoAction;
        }
        size = 1024;
        default_action = drop();
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
