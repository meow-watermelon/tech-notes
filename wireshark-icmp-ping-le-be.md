# BE / LE Values in WireShark ICMP Ping Packets

## What is it?

The sequence number field is simply being displayed in both big endian (BE) and little endian (LE) formats to make it easier to follow when those sequence numbers are incrementing from one ICMP echo request/reply to the next. The reason both formats are shown is because sometimes those fields are populated in big endian format and sometimes they are populated in little endian format, and there is no definitive way to tell which format it's in from the contents of the packet.
