# P4 Tutorial
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

