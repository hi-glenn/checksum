# checksum


This is a reconstruction of http://www.roman10.net/2011/11/27/how-to-calculate-iptcpudp-checksumpart-2-implementation/
The site is gone so I took it from the archive for the sake of [C Programming TCP Checksum's answer](https://stackoverflow.com/a/51269953/1346426)
Enjoy! All credits go to https://github.com/roman10.

This is a follow up of the previous post, how to calculate IP/TCP/UDP checksum part 1 — theory.

# IP Header Checksum Calculation Implementation
To calculate the IP checksum, one can use the code below,

```c
/* set ip checksum of a given ip header*/
void compute_ip_checksum(struct iphdr* iphdrp){
  iphdrp->check = 0;
  iphdrp->check = compute_checksum((unsigned short*)iphdrp, iphdrp->ihl<<2);
}
/* Compute checksum for count bytes starting at addr, using one's complement of one's complement sum*/
static unsigned short compute_checksum(unsigned short *addr, unsigned int count) {
  register unsigned long sum = 0;
  while (count > 1) {
    sum += * addr++;
    count -= 2;
  }
  //if any bytes left, pad the bytes and add
  if(count > 0) {
    sum += ((*addr)&htons(0xFF00));
  }
  //Fold sum to 16 bits: add carrier to result
  while (sum>>16) {
      sum = (sum & 0xffff) + (sum >> 16);
  }
  //one's complement
  sum = ~sum;
  return ((unsigned short)sum);
}
```

The method compute_ip_checksum initialize the checksum field of IP header to zeros. Then calls a method compute_checksum. The mothod compute_checksum accepts the computation data and computation length as two input parameters. It sum up all 16-bit words, if there’s odd number of bytes, it adds a padding byte. After summing up all words, it folds the sum to 16 bits by adding the carrier to the results. At last, it takes the one’s complement of sum and cast it to 16-bit unsigned short type. You can refer to part 1 for more detailed description of the algorithm.

Note that the data structure iphdr and tcphdr and udphdr are Linux data structures representing IP header, TCP header and UDP header respectively. You may want to Google for more information in order to understand the code.

# TCP Header Checksum Calculation Implementation
To calculate the TCP checksum, you can use the code below,

```c
/* set tcp checksum: given IP header and tcp segment */
void compute_tcp_checksum(struct iphdr *pIph, unsigned short *ipPayload) {
    register unsigned long sum = 0;
    unsigned short tcpLen = ntohs(pIph->tot_len) - (pIph->ihl<<2);
    struct tcphdr *tcphdrp = (struct tcphdr*)(ipPayload);
    //add the pseudo header 
    //the source ip
    sum += (pIph->saddr>>16)&0xFFFF;
    sum += (pIph->saddr)&0xFFFF;
    //the dest ip
    sum += (pIph->daddr>>16)&0xFFFF;
    sum += (pIph->daddr)&0xFFFF;
    //protocol and reserved: 6
    sum += htons(IPPROTO_TCP);
    //the length
    sum += htons(tcpLen);
 
    //add the IP payload
    //initialize checksum to 0
    tcphdrp->check = 0;
    while (tcpLen > 1) {
        sum += * ipPayload++;
        tcpLen -= 2;
    }
    //if any bytes left, pad the bytes and add
    if(tcpLen > 0) {
        //printf("+++++++++++padding, %dn", tcpLen);
        sum += ((*ipPayload)&htons(0xFF00));
    }
      //Fold 32-bit sum to 16 bits: add carrier to result
      while (sum>>16) {
          sum = (sum & 0xffff) + (sum >> 16);
      }
      sum = ~sum;
    //set computation result
    tcphdrp->check = (unsigned short)sum;
}
```

The method sums the pseudo TCP header first, then the IP payload, which is the TCP segment. It also pads the last byte if there’re odd number of bytes. For detailed description of the algorithm, please refer to comments in the code and part 1.

# UDP Header Checksum Calculation Implementation
To calculate the UDP checksum, one can follow the code below,

```c
/* set tcp checksum: given IP header and UDP datagram */
void compute_udp_checksum(struct iphdr *pIph, unsigned short *ipPayload) {
    register unsigned long sum = 0;
    struct udphdr *udphdrp = (struct udphdr*)(ipPayload);
    unsigned short udpLen = htons(udphdrp->len);
    //printf("~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~udp len=%dn", udpLen);
    //add the pseudo header 
    //printf("add pseudo headern");
    //the source ip
    sum += (pIph->saddr>>16)&0xFFFF;
    sum += (pIph->saddr)&0xFFFF;
    //the dest ip
    sum += (pIph->daddr>>16)&0xFFFF;
    sum += (pIph->daddr)&0xFFFF;
    //protocol and reserved: 17
    sum += htons(IPPROTO_UDP);
    //the length
    sum += udphdrp->len;
 
    //add the IP payload
    //printf("add ip payloadn");
    //initialize checksum to 0
    udphdrp->check = 0;
    while (udpLen > 1) {
        sum += * ipPayload++;
        udpLen -= 2;
    }
    //if any bytes left, pad the bytes and add
    if(udpLen > 0) {
        //printf("+++++++++++++++padding: %dn", udpLen);
        sum += ((*ipPayload)&htons(0xFF00));
    }
      //Fold sum to 16 bits: add carrier to result
    //printf("add carriern");
      while (sum>>16) {
          sum = (sum & 0xffff) + (sum >> 16);
      }
    //printf("one's complementn");
      sum = ~sum;
    //set computation result
    udphdrp->check = ((unsigned short)sum == 0x0000)?0xFFFF:(unsigned short)sum;
```

The code is similar to TCP checksum computation. Except that when the checksum is compted as all 0s, we set them to 1s. As 0x0000 is already reserved for indicating that the checksum is not computed. Also please refer to part 1 for detailed description of the algorithm.

All the code can be downloaded at part 3.
