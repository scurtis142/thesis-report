###ABSTRACT

###INTRODUCTION

###BACKGROUND KNOWLEDGE

###PREVIOUS WORKS

###METHODOLOGY / METHOD / EXPERIMENT

##Overview
   XXXTODO. what was achieved overall

   The general format of the experiment was set up with 2 machienes, one acting as a network
   simulator, which has the task of generating network traffic for the experiment, and the other 
   was the test macheine, which ran the experiments and recorded results. 

   This of course is not the only way that the experiment could have been performed. For instance,
   two 10Gbit network cards (or even a single one with two ports, such as the X540-T2) could be set
   up on the same machiene, using one port for transmission and the other for recieving. Another
   idea could be to use virtual machienes as the transmitting and recieving deviecs. However, both
   of these ideas introduce hidden variables that might not be taken into account when performing
   the experiment. For example, one of the systems that exists in computers and is part of the
   pipeline from network wire to cpu is the bus speed. Although this is not normally an issue, it
   still has a maximum bandwidth. For example the Network card that is used in the experiment has a
   8 lane pci speed of 5.0GT/s. Although this is definatly enough to handle 10Gbits/s, adding two
   ports to a single pci slot is one extra thing that could become a potential bottleneck, so it is
   best to avoid things like this if possible. 

   Of course another problem of using dual ports on a single machiene or virtual machienes is that
   the transmitting and recieving processes now have to content for cpu resources. All of these
   factors are things that we would want to minimise while testing. Having seperate machiens also
   closely mimics a real world network, another reason why it was chosen as the test setup.

   One of the first problems encountered while designing the experiment was the problem of how to
   generate substantially fast network traffic. Since we did not have a real life 10Gbit/s+ network
   to test on, the author had to come up with a way to generate this amount of traffic with only the
   materials avaliable to him. (Also, it probably wouldn't be wise to perform this experiment on a
   real network, as the results would be inconsistant. e.g. you wouldnt have consistantly sized
   packets, and the network might not always be running at one particular speed.). The original
   problem stated in this report is that a Linux kernel is not capable recieving 10Gbits/s traffic.
   This is later shown in the results. So from this, we can also safely presume that it is not able
   to generate high speed traffic either. Expensive and purpose build hardware might be possible in
   this situation, but due to budget constraits this was not possible in this case. Another soultion
   might have been to generate traffic from multiple machienes and forward and funnel to to the test
   machiene. Again, this would have needed more physical hardware, as well as a switch that could
   forward the traffic to the test machiene. Also this would have involved considerably more
   communication and co-operation and timing between the seperate network simulators. 

   The option that remained was to use a single machiene, but use some software that was capable of
   doing what was required. Several packet generator tools were tested out, but there were very few
   that gave the required speed, were free, and had sufficient functionality. Eventually a packet
   generation tool "pktgen" was selected. This is a packet generation tool that is built on top of the
   DPDK framework. This raises the obvious question of: 'If the packet generation tool is build upon
   DPDK, then isn't that going to be a limiting factor during testing'. The answer is yes, the
   maximum bandwidth that we can test for is limited by the generation framwork, which is also
   something we are testing. However, from the results we can see that this tool is definalty
   capable of generating 10Gbit/s traffic, which was the goal of this experiment. Also it is worth
   noting that the pktgen tool will only be generating packets and sending them off. The recieving
   software will have the added cost of implementing a working NetFlow table on top of this. This is
   the best solution that the author could come up with, but it should be noted for future work that
   the task of packet generation should be delagated to a tool that is not also being tested, if the
   budget allows. 

   DPDK's 'pktgen' tool is capable to generating 10Gbit wire rate traffic with 64 byte frames. It
   has a runtime environment to configure, start and stop flows, and can diplay real time metrics.
   To run pktgen, you first have to install dpdk on your simulator machiene. Be aware that dpdk has
   certain bios, system and toolchain requirements for it too work properly. The critical specs for
   the pc that was used to simulate traffic are:
      OS: Ubuntu Linux: 4.15.0-101-generic
      CPU: 
      Memory:
      NIC: X540-T1
   For the full list of specs please see the Appendix. Note a critical requirement to run dpdk is to
   have the sse4_2 cpu flag enabled. You also need at least 2 cores but preferable 4. A core i7 or
   better is recommended.  

   Once pktgen was running the method to perform the tests was:
      1. set the size of the frames to be sent with the command: 
            set 0 size [size in bytes]
      2. set the desired traffic rate with:
            set 0 rate [rate in percent]
      3. set the number of packets to be sent with: 
            set 0 count [number of packets]
      3. when the test machiene is ready to receive, start the transmission with:
            start 0
      4. once the current transmission rate drops to zero, (signifying the end of the test), stop
         the port with:
            stp
      5. to verify the correct bytes and packets were sent, run:
            page stats
   
   The 0's in the above command signify configuring the 0th port. (The only port in this case).
   Below we will identify how the values for size, rate and count were calculated.

   The size parameter is just iterated through the different packet sizes that were tested in this
   experiment i.e. iterated through {64, 128, 256, 512, 1024}. The rate parameter was iterated
   through the different rates that were tested (as a percent of 10Gbit/s) i.e. {1, 10, 50, 75, 100}
   or (100Mbit, 1Gbit, 5Gbit, 7.5Gbit and 10Gbit) respecivly. This is because pktgen calculates the
   maximum rate it can send on the configured network card, and takes the rate as a percentage of
   this. The count parameter is calcuated from the rate and the size. Specifically, take the rate
   and calculate the number of bits that will be sent after transmitting for 10 seconds at that
   rate. (All tests were performed over 10 second periods). Then divide this by the 8 to get the
   number of bytes, and divide again by the packet size to get the number of packets.

   For running the multiple flow tests, pktgen was configured to read and replay from a pcap file.
   This pcap file was a excerpt of an irc exchange, as well as some other background flows. To
   ensure that pktgen would run at the correct rate, it is critical to remove all the timestaps from
   the pcap file, otherwise pktgen will add these delays in. The same commands as before can be used
   to start the tests, so once pktgen is running and the test machiene is ready to receive, start
   the tests with:
      1. set 0 rate [rate in percent]
      2. start 0. 
   At the same time as calling the start command. Start a timer for 10 seconds. Then run the stop
   commmand.
      3. stp
   Although this may seem like it introduces variance by manually starting and stopping the timer.
   It is not a major issue as you can still read the number of sent packets and bytes and use this
   to calculate what result you should be seeing on the test machiene, and how much byte and packet
   loss has occoured. You can do this with
      4. page stats
   For example, after running page stats you may see that the number of bytes sent is _____ and the
   number of packets is _____. Comparing these to the results from the test machiene for the number
   of bytes and packets received of ___ and ___, you can conclude that the packet and byte loss for
   that test is ___ and ___.

   To make this process of running test simulations more efficiant, you may want to utilise pktgen's
   scripting functionality via the Lua programming language. 


##TEST METHODOLOGY

   Below is described what happens on the test machiene. 
   
   The process to describe what happens on the test machiene can be split into 4 sections.
   (capture framwork, base process, netflow table, exporter, visual tool). The majority of the code
   written and used in this experiment is in C. There was a little bit of python used for the
   visualisation tool, and some shell scripting for automating testing and setup.

   Firstly, there is the capture framework library. Included in here is the library functions, api,
   drivers and kernel modules for the capture frameworks that were used (dpdk, netmap, pf_ring_zc).
   For the default Linux kernel tests, this can be thought of as the kernel and system call
   interface. This is the code that is tasked with reading the packets off the nic, either by
   polling the interface, or via interrupts / signals. This is an essential part of what we are 
   testing in this experiment, i.e. how efficiant and fast these frameworks are at network IO. 

   Although we do not directly modify the library framwork code, we do use the systems and
   interfaces that they provide in our program. For example, we will call all initialisation
   functions neccesary to set up the capture framworks and will use their provided api for getting
   access to packet header and date pointers. 

   As an example in each of the specific frameworks: 

   For the dpdk implementation we had to call: rte_eal_init, to set up the run time enviroment,
   rte_eth_dev_count_avail; to find the number of avaliable port, rte_pktmbuf_pool_create; to create
   the memory buffer pool, setup ports with rte_eth_rx_queue_setup, setup all thread needed,
   initialise the netflow table etc. The main loop with gets pointers into avaliable packets looks
   like: 

   while (!force_quit) {
      for (i = 0; i < nr_queues; i++) {
         nb_rx = rte_eth_rx_burst(port_id,
               i, mbufs, 32);
         if (nb_rx) {
            packet_classify_bulk (mbufs, nb_rx, table);
            for (j = 0; j < nb_rx; j++) {
               struct rte_mbuf *m = mbufs[j];
               rte_pktmbuf_free(m);
            }
            pkt_cnt += nb_rx;
         }
      }
   }

   where 'packet_classify_bulk' is the function that does the packet process and eventually update
   the netflow table, although this section of the code will be discussed in a later section.

   For Netmap the main library functions that are called include: nmport_prepare, parse_nmr_config,
   nmport_open_desc. The main loop for receiving packets is: 

   for (i = targ->nmd->first_rx_ring; i <= targ->nmd->last_rx_ring; i++) {
      int m;

      rxring = NETMAP_RXRING(nifp, i);
      /* compute free space in the ring */
      m = rxring->head + rxring->num_slots - rxring->tail;
      if (m >= (int) rxring->num_slots)
         m -= rxring->num_slots;
      cur_space += m;
      if (nm_ring_empty(rxring))
         continue;

      m = receive_packets(rxring, targ->g->burst, dump, netflow, targ->g->n_table, &cur.bytes);
      cur.pkts += m;
      if (m > 0)
         cur.events++;
   }

   where the recieve_packets function takes a pointer to netmap's ring buffer. This function
   eventually inserts an entry into the flow table. 

   The main program for the pf_ring implementation looks similar. We start by calling the
   pfring_zc_create_cluster library function. The device is opened with the pfring_zc_open_device
   function and the licensce is checked with the pfring_zc_check_device_license function. The main
   packet consumer thread gets a pointer to the next oacjet by calling pfring_zc_recv_pkt_burst.
   This pointer is then passed to the get_netflow_k_v function to get a key value entry and then
   these are passed to the netflow_table_insert function to be inserted into the hash table
   (explained in more detail below).

   The linux kernel implimentaion work by using the raw socket interface that the kernel provides.
   The main system calls that are needed to for this setup are the 'socket' system call, the ioctl
   system call to set the interface to promisuious mode and the 'recvfrom' system call to get a
   pointer to the next packet. Again, this is then passed to the relavent NetFlow table functions. 
   
   As part of the cleanup that happens when the process receives the interrupt signal telling it to
   close, it prints the total packet, byte and flow counts of all the entries in the flow table. It
   does this by looping over each list in each entry of the hash table. The reason for this is
   explain in more detail in a later section, but this is how the throughput into the table and the
   percent packet loss is measured.


##NETFLOW TABLE DESIGN

   The NetFlow table that was implemented for use with the different framworks was designed with a
   couple of contraints in mind. Firstly, it needed to be fast to insert and update flow entries.
   This should be obvious as performance is the primaty metric of this report. It also needs to
   make sure it is fast as to not affect the performance of the capture framework. If it takes too
   long to insert an entry then the framework being tested might drop packets that it otherwise 
   could have read. Another constraint was that it had to be as consistant as possible when used 
   across the different frameworks. This is to ensure that the comparison between the framworks is
   as fair as possible. However there are some exceptions to this case. Ntop, the organisation that
   created the pf_ring_zc library, also have a native flow table implementation using the pf_ring_zc
   table. Since useing and testing this table turned out to be not a lot of work, an analysis of
   this table was included in this report, but it should be noted that this result was not using the
   same flow table as the other frameworks. Another exeption was the dpdk flow table implementation,
   which I was able to modify from an existing code base. The code in this section has a few extra
   things that were not included in the flow tables for the other frameworks. Most of the
   differences are negligable, but it is worth noting one change that might be significant. The DPDK
   flow table was able to utilise hardware hashing functions whereas the other table implementation
   used software. It is assumed that the hardware hashing functions were faster than software, but
   it was not investigated into how much difference this made. Most other differences between the
   design of the flow table were minor and posed no significance. A third requirement of the flow
   table was that is it safe for multiple threads. This is due to the 'exporter' thread that
   periodically flushes the table to do something with the data. It is not a good idea that this
   thread has access to the table whilst other threads are inserting or updating entries. Similarly,
   it is important that only one thread has the capability to update the table at one time. 

   To conform to these requirements the NetFlow table was implemented as a hash table. This has the
   the properties of constant inserting time (as long as the number of flows is not greater than
   half the size of the table *referance*). Note that in the case of the tests that were run in this
   experiment, it was always the case that inserting into the table took constant time, as the
   number of flows in the test cases were always less that the number of Flow table entries, which
   is 1024 by default. In a real-world scenario, there may be more active flows than table entries.
   In this case, flows would double up, with multiple flows mapping to a single table entry. This
   is expected, and would not corrupt the data as the flows would be seperated in a list structure,
   however, performance may begin to slow. However if optimal performace is still required with a
   high number of flows, the user could easily modify the default start size of the table to be
   whatever they desire, as long as computer resources permit. 
   
   The hashing function that is used to map flows to their corresponding hash table entries is a
   crc32 checksum of the 5-tuple fields described by a NetFlow 'flow'. This checksum is provided by
   hardcoded hashing tables in software, and by hashing recursivly passing each field of the flow
   through the checksum. 

   idx = crc32c_1word (key->proto, idx);
   idx = crc32c_1word (key->ip_src, idx);
   idx = crc32c_1word (key->ip_dst, idx);
   idx = crc32c_1word (key->port_src, idx);
   idx = crc32c_1word (key->port_dst, idx);
   idx = idx % table->n_entries;

   As you can see by generating the table index using each of the fields in the flow, we can get a
   pseudo-unique index for each flow.  It is also worth noting how the "key" struct is populated.
   This is done in the 'get_netflow_k_v' function that takes a pointer to the packet header, and
   then traverses, 

   Once we have the index it is a simple matter of navigating to the offset and traversing the list
   (which we expect to have a length of 1 for a low number of flows), then updating the byte and
   packet counters for that flow. 
   

##TABLE EXPORTER FUNCTION

   Normally a NetFlow exporter would send expired flow data to a third party collection service. For
   our table implementation, it was decided that it would be better to test the implementaion of a
   Flow table a more simply way, by simpily periodically writing out the flow table as a csv file.
   There were several reasons for this. Firstly, it was for simplicity reasons. Although the process
   did not act like an exact or complete NetFlow exporter, it could be adapted to easily by simply
   changeing what the exporting thread does. So the proof of concept is there. Another reason is
   that it makes un-neccesary work to go and set up some flow collection service, as it would not
   add anything to the project and is not within the project's scope.

   The process of going exporting the flow table to a csv file was quite simple. Since the flow
   table was designed to be thread safe (see section above), a exporting thread was set up to run
   with a period of 1 second. This thread simply looped through each list in the hash table, copying
   the flow information, along with the current packet and byte count to a temporary buffer,
   seperated by commas. Once all the entries had been added to this buffer, the lock on the flow
   table is released and the buffer is written out to a file. It is important that a temporary
   buffer is used and there is only 1 write operation so that the exporting does not hold up the
   task of inserting flows. Since writing is an IO operation and quite time expensive (relative to
   other computer operations), if multiply write's are done and the inserting thread has to wait for
   these to finish then it has to potential to drop a significant amount of packets as the
   underlying frameworks packet buffers fill up. It is also worth noting that the exporting thread
   does not do any processing of the flow data, such as sorting or filtering, for the same reason
   that it would be a bad idea to do any un-nessesary work that may hold up other critical threads.
   This sort of processing work is handled by the program that reads from the exported file, and is
   described in the next section.  

##FLOW VISUAL TOOL

   To add a visual and practical element to this experiment, a real time flow visualiser tool was
   implemented to test the code for the flow table in it's ability to export data, to make sure what
   is was storing is correct, and to add a use case for this experiment. Although a normal NetFlow
   exporter will export flows via the network in to some third party collection software, our
   implementation took a simplified appreach. Instead of creating a full blown NetFlow exporter, the
   NetFlow table simplify exported itsself to a temporary csv file, once per second (see section
   above for more details). This was a flexible approach that allowes any other process to access
   this data and use it as it wishes. For example, a process could read the file and then export the
   flows via the network to a third party collector, or (as in our case) implement a live table with
   packet and byte counts for the 10 biggest flows. 

   The visual tool is written in python and simply opens the exported csv file, sorts the entries by
   the number of bytes and then displays a table to the top 10 biggest flows. Doing this in a loop
   every 1 seconds gives a live updating table. 

   Source IP         Destination IP    Source Port   Destination Port   Protocol   Packets     Bytes

   192.168.1.2       212.204.214.114   2848          6667               tcp        7124        398460
   212.204.214.114   192.168.1.2       6667          2848               tcp        5862        4852461
   192.168.1.2       71.10.179.129     4026          14232              tcp        1939        111254
   71.10.179.129     192.168.1.2       14232         4026               tcp        1939        160889
   172.200.160.242   192.168.1.2       11352         4984               tcp        1849        153177
   192.168.1.2       172.200.160.242   4984          11352              tcp        1849        105001
   192.168.1.2       68.206.150.243    1312          57322              tcp        1280        79794
   24.177.122.79     192.168.1.2       8022          3863               tcp        1213        81276
   192.168.1.2       24.177.122.79     3863          8022               tcp        1213        75429
   192.168.1.2       67.71.69.121      1092          12492              tcp        903         43903

##COLLECTING RESULTS

   The format for running the tests and collecting the results followed a methodical procedure.
   Firstly, the correct packet count, size and offered throughtput were calculated for the packet
   generation tool (pktgen). This is explain in more detail in section xxx. The setup script for the
   particular framwork being tested is run, and then if there are no problems, the start script is
   then run. This means the program is not collecting packets and populating the flow table. Then 
   the start command is given to pktgen. The network test is now in progress. Once pktgen shows that
   the bandwidth has dropped back down to 0, the stop and page stats commands are issued and an
   interrupt signal is sent to the main testing program. The stats for the number of packets sent
   and the number of bytes sent can be read from the stats page of pktgen. The test program should
   have outputted a sum of the total number of bytes and a total number of packets in all of the
   flows stored in the table. This way of measuring ensures accuracy that the number of packets and
   bytes recoreded at the test machiene have made it all the way through the test procedure, rather
   than just being read from the interface, which would be incorrect. 

   One more calculation has to be done before however, because pktgen and the test process may
   calculate the number of bytes differently. Specifically, if packet gen is given a packet size of
   x bytes, it will actually count that packet as (x-4) bytes. As the 64 bytes includes an option
   VLAN field sent in the ethernet header. Since this field is set to null in pktgen, it is not
   included and the actual size of the frame is 60 bytes. On the other end of the test setup, the
   ethernet header actually get stripped of before it is read by the NetFlow table code, and the
   table caluclates the packet size from the ip header field. To verify the integrity of the above
   statements, one only needs to set up an expirement where pktgen sends only 1 packet, and then
   read the corresponding sent and received stats. Another difference is in the native flow table
   for pf_ring_zc, which actually include the preamble and crc check into byte counts as well. To 
   counter this inconvinance, the results table has to contain 'expected bytes' field, in which
   takes as an input the number of bytes sent as read from the pktgen stats, and output the expected
   number of bytes to compare against.  

   Then the difference between the expected bytes and packets and the bytes and packets recieved may 
   be used to calculate the measured throughput and percent packet loss fairly. 

##OTHER NOTES
   any other final notes with regards to the methodology (look through paper notes)

###RESULTS

###DISCUSSION AND FUTURE WORK
