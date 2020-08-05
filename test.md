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
    macAddr_t dstAddr;
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

header_type ipv6_t {
    fields {
        bit<4> version;
        bit<8> trafficClass;
        bit<20> flowLabel;
        bit<16> payloadLen;
        bit<8> nextHdr;
        bit<8> hopLimit;
        bit<128> srcAddr;
        bit<128> dstAddr;
    }
}

header_type udp_t {
    fields {
        bit<16> srcPort;
        bit<16> dstPort;
        bit<16> hdrLength;
        bit<16> chksum;
    }
}

header_type tcp_t {
    fields {
        bit<16> srcPort;
        bit<16> dstPort;
        bit<32> seqNum;
        bit<32> ackNum;
        bit<4> dataOffset;
        bit<6> unused;
        bit<6> flags;
        bit<16> windowSize;
        bit<16> chksum;
        bit<16> urgPtr;
    }
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

parser parse_ethernet {
   packet.extract(hdr.ethernet);
    transition select(hdr.ethernet.etherType) {
        0x0800    : parse_ipv4;
        0x86DD    : parse_ipv6;
        default   : accept;
    }
}

parser parse_ipv4 {
    packet.extract(hdr.ipv4);
    transition select(hdr.ipv4.protocol){
        0x11      : parse_udp;
        0x06      : parse_tcp;
        default   : accept;
    }
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
