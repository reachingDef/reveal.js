Outline
1. Motivation & Hypothesis
2. TLS protocol background
3. TLS library
4. Instrumentation
5. Experiment(s)
6. Results
7. Constraints / drawbacks
8. Outlook

--

Motivation
* IOT, Industry 4.0, cyber physical systems..
* huge number of: <!-- .element: class="fragment" data-fragment-index="1" -->
* resource constrained devices<!-- .element: class="fragment" data-fragment-index="1" -->
* connected to each other<!-- .element: class="fragment" data-fragment-index="1" -->
* bonus: real time requirements<!-- .element: class="fragment" data-fragment-index="1" -->

--

Question: What about sensitive data? / How to protect the communication?

--

Hypothesis: TLS is suited for the protection of low-resource realtime systems

--

Why (not) TLS?
*  well known protocol
*  many implementations
*  considered to be slow <!-- .element: class="fragment" data-fragment-index="1" -->
*  and resource hungry <!-- .element: class="fragment" data-fragment-index="1" -->

--

objectives:
* low resource = limited RAM, persistent storage and computational capacity
* realtime = fixed (and low!) latency

--

latency
* roughly: time between call of TLS library function until system IO function is reached
* channel establishment<!-- .element: class="fragment" data-fragment-index="1" -->
* exchange of user data  <!-- .element: class="fragment" data-fragment-index="1" -->

--

Verification in two steps:
1. development of a test bench <!-- .element: class="fragment" data-fragment-index="1" -->
2. evaluation of TLS on selected target platforms <!-- .element: class="fragment" data-fragment-index="2" -->

--

Brief background on TLS

--

<img  width="800"; height="auto"; src="images/tls_layers.svg"/>
* relevant sub protocols: Handshake & record layer protocol 

--

<img height="600"; width="auto"; src="images/SSL_handshake.png"/>

--

<img src="images/MS_CS_dissection.png"/>

--

Why is the choice of cipher suite so important?
* one of the very few dynamic components
* cryptographic operations dominate the runtime
* test all cipher suite for influence on latency

--

# test bench
1. tls implementation
2. instrumentation
3. (now that we have the data..) comm protocol
4. software architecture (components)
5. correctness

--

Goal

* give user simple metrics by which cipher suites can be chosen
* provide tools to allow for deeper insights where necessary

--

TLS implementation
* OpenSSL (and derivates): too heavy <!-- .element: class="fragment" data-fragment-index="1" -->
* wolfSSL: suited for embedded environments, seems somewhat "hacky" <!-- .element: class="fragment" data-fragment-index="2" -->
* mbed TLS: ARM's official TLS library, supports almost all TLS features, felt convenient to use <!-- .element: class="fragment" data-fragment-index="3" -->

Note:
* SSL2.0: 1995
* TLS 1.0: 1999
* TLS 1.2: 2008

--

Instrumentation

--

How to instrument?
* callgrind: excellent tool, just too heavy
* gprof: had a lot of quirks, also not sure if available on target platforms
* considered other papers and their approach, none really suited mine

--

Solution: home brew instrumentaion
* small library linked to target code
* manual insertion of log points into the code
* relies on clock_gettime call

--

API
* label, time stamp and (optional) payload
* embracing log points (i.e. start and end label)
* Usage 
        int my_function(...) {
            log_point(MY_FUNCTION_START, global_log_ctx, 0);
            // do stuff
            log_point(MY_STUFF_START, global_log_ctx, 0);
            // do stuff
            log_point(MY_STUFF_STOP, global_log_ctx, 0);
            // do clean-up stuff
            log_point(MY_FUNCTION_STOP, global_log_ctx, 0);
            return result;
        }

Note:
* nested logs possible (and heavily used)

--

Log trees
<img src="images/entries_to_tree.png"/>
* nested log points form a log tree
* similar to matching many different parenthesises

--

Completeness
* assume single-threaded code
* target code can be surrounded by outer log label
* then complete execution time (plus noise) is captured
* nesting log points increases preciseness

--

Where to place the log points?

--

structure of mbed
* gained insight into library via callgrind (and KCachegrind)
* split into three libraries: crypto, tls, x509
* TLS protocols steps each have their own function 
* user provides network I/O callbacks <!-- .element: class="fragment" data-fragment-index="1" -->

Note:
* callback: we're able to exclude time spent in kernel for I/O

--

placement of logs
* cover protocol step functions
* cover the whole crypto API
* idea: most time is spent in (a)symmetric crypto operations
* knowledge of protocol step sufficient, no general need for further insight

--

covering the crypto API
* mbed TLS comes with a built-in benchmark tool that measures raw crypto performance
* went through list of included header files to identify roughly all crypto functions
* perks: by instrumenting the function declaration and not explicit calls, all calls are
implicitly instrumented
* crypto functions considered to be "primitive" (no need to look insight them, impl. irrelevant)

--

Now we can instrument the implementation locally. What's next?

Let's take a look at the "other" side of the test bench!  <!-- .element: class="fragment" data-fragment-index="1" -->

--

Components of the test bench
<img src="images/components.png"/>

Note:
* driver: runs the experiment code 
* built by builder
* controller: orchestrates the experiments

--

<img height="600"; width="auto"; src="images/SetupSketch.svg"/>

Note:
* physical setup: TWR board, powerful tower PC and laptop connected through 1Gb ethernet
* bare bone linux installation
* board and tower run driver application, laptop controller
* virtual links: drivers speak TLS, controller and drivers use custom protocol

--

<img height="600"; width="auto"; src="images/ProtoInitial.png"/>

--

experiment 
* controller sends current CS to client
* (number of iterations, total payload size, size of a single packet)
* here (1000, 10000, 1000)
* since TLS asymmetric protocol: roles change

Note:
* read: each cipher suite was measured 1000 times. During each iteration,
a total of 10000 bytes was exchanged, where each packet was 1000 byte large

--

<img height="600"; width="auto"; src="images/ProtoNormal.png"/>

Note:
* CLI_INS_HANDSHAKE has parameter to enforce cipher suite

--

<img height="600"; width="auto"; src="images/ProtoBufferFull.png"/>

--

Internally
* client initiates and proceeds handshake 10 times
* after each channel establishment, 10x 1KB dummy payload is sent from client to server
* repeat after all cipher suites were used
* then start again (total: 100 iterations)
* reason: whole experiment for single CS takes very long. By splitting up,
already measured logs can be processed. Also, an "open end" scenario is thinkable

--

# Evaluation
1. Methodology
1. results
2. overhead 
3. drawbacks

--

Methodology
* channel establishment with enforced cipher suite
* always: initiator ("client") sends 10 packets with 1kB each to responder ("server")
* I/O: refers to network I/O (not RAM / persistent storage)

--

Results
* latencies for channel establishment and user data exchange
* user data exchange split into symmetric cipher and hashing algorithm
* detailed view of time protocol steps take, in chronological order
* overview of how much time different types of operation take in protocol steps

--

* "handshake": runtime of 'mbedtls_ssl_handshake' function
* "user data exchange": runtime of 'mbedtls_ssl_read' / 'mbedtls_ssl_write' function

--

<img src="images/w_ov_by_keychg.svg"/>
Note:
* combined overview of handshake and user data exchange latencies
* y-axis: median time of 1000 handshakes (init / respond)
* x-axis: median time of 10x1000 user data exchange (1 kByte) (write / read)
* nice separation into layers parallel to x-axis
* bottom: PSK
* performance asymmetry: RSA performs well (client only encrypts)
* elliptic curves with average performance
* slowest: 'pure' DH

--

<img src="images/w_ov_by_sym.svg"/>
Note:
* separation into layers parallel to y-axis
* RC4 fastest (broken, but stream ciphers interesting, Salsa20)
* fastest secure cipher: AES 128 CBC
* 3DES unsurprisingly slowest
* same for handshake responder side, reading side - omitted due to time constraints

--

Winners
<table>
<tr>
<td>read</td><td>write</td><td>handshake initiator</td><td>handshake responder</td>
</tr>
<tr>
<td>69.92us</td><td>67.36us</td><td>1159.84us</td><td>1265.88us</td>
</tr>
<tr>
<td>RC4-128-MD5</td><td>RC4-128-MD5</td><td>PSK-WITH</td><td>PSK-WITH</td>
</tr>
</table>

Note:
* CS fixed set, no overall winner

--

Recommendation
* read/write: AES-128-CBC (AES-128-GCM if AES hardware accelerated)
* handshake initiation: PSK (if infrastructure available), RSA 
* handshake responder: PSK (if infrastructure available), RSA
* avoid 3DES and DH(E)
* reasonable choice: TLS-RSA-WITH-AES-128-GCM

--

Showing the single protocol step of a handshake for a single cipher suite

--

<img src="images/49320-stacked_COMPLETE_HANDSHAKE_Board_exc_io.svg"/>
Note:
* idea: "Where exactly is time spent during the handshake?"
* outer bar: total handshake time
* inner bars: protocol steps
* y-axis: linear time in micro seconds
* steps in chronological order
* 1000 iterations, median time of each protocol step
* presentational approach can be used for any kinds of labels, also arbitrarly deep

--

Overview of the internals during a handshake (cipher suite: 147)

--

<img src="images/groupby_COMPLETE_HANDSHAKE_Board_exc_io_147.svg"/>
Note:
* idea: "Which types of cryptographic operation is most time spent on?"
* here for handshake, also possible for user data exchange
* client side
* most time spent on asymmetric cryptography 
* RNG: random number generator (secret generation)
* Rest: time in handshake block that's not been covered by primitive crypto labels

--

<img src="images/groupby_COMPLETE_HANDSHAKE_Board_inc_io_147.svg"/>
Note:
* for reference, how much time is spent in I/O?
* I/O: TLS, I/O logs: transportation of logs
* really makes sense to exclude I/O time

--

A closer look on latency (types)

--

<img height="350"; width="auto"; src="images/write_latency_expl.svg"/>
* latency 1 = L2 − L1 − IO <!-- .element: class="fragment" data-fragment-index="1" -->
* latency 2 = L3 − L1 − IO <!-- .element: class="fragment" data-fragment-index="1" -->
* latency 3 = L3 − L1 <!-- .element: class="fragment" data-fragment-index="1" -->

Note:
* mbedtls_ssl_write function
* no knowledge beforehands where something TLS related is done and where I/O is involved
* latency: extra time spent in TLS layer
* question: from where to compute start / end?
* differentiate by: include post-processing? include IO?

--

<img height="350"; width="auto"; src="images/read_latency_expl.svg"/>
* latency 1' = L3' − L1' − IO<!-- .element: class="fragment" data-fragment-index="1" -->
* latency 2' = L3' − L2' − IO<!-- .element: class="fragment" data-fragment-index="1" -->
* latency 3' = L3' − L2'<!-- .element: class="fragment" data-fragment-index="1" -->

Note:
* stop is fixed, 

--

<img src="images/5-Board_rw_dens.svg"/>

--

<img src="images/5-Board_rw_hist.svg"/>

--

Consider hardware accelaration for cryptographic ciphers..

--

<img src="images/global_groupby_DO_SSL_WRITE_Board_exc_io_by_hash.svg"/>

--

<img src="images/global_groupby_DO_SSL_WRITE_Board_exc_io_by_sym.svg"/>

--

performance impact
* measured on two levels
* micro: run two crypto functions (slow, fast) with different forms of logging
* macro: compare the runtime of an instrumented version of mbed TLS with an instrumented

Note:
* different forms = no logging, logging with metric function, two different clock modes

--

micro
* mean of N=1000000 runs
<table>
<tr>
    <td>Operation</td>
    <td>Raw (ns)</td>
    <td>Monotonic / Raw</td>
    <td>Monotonic Coarse / Raw</td>
</tr>
<tr>
    <td>SHA256</td>
    <td>1266.5ns</td>
    <td>1.114</td>
    <td>1.090</td>
</tr>
<tr>
    <td>3DES</td>
    <td>8799.2ns</td>
    <td>1.012</td>
    <td>1.007</td>
</tr>
</table>

--

macro
* instrumentation still takes place in bench process
* no deeper insight possible (IO ?)
* compare handshake and read and write latency
* however: time used to transmit logs captured

--

overhead, pc as server, board side
<img src="images/global_difference_write_latencies_by_keyxchg_PC_AS_SERVER.svg"/>

--

overhead sym, pc as server, board side
<img src="images/global_difference_write_latencies_by_sym_PC_AS_SERVER.svg"/>

--

overhead, pc as server, pc side
<img src="images/global_difference_read_latencies_by_keyxchg_PC_AS_SERVER.svg"/>

--

overhead sym, pc as server, pc side
<img src="images/global_difference_read_latencies_by_sym_PC_AS_SERVER.svg"/>

--

portability and flexibility:
* benchmark code written in C, very little outer dependencies
* other clock functions may be used
* theoretically also other metrics, e.g. energy consumption
* requirements on metric function: monotonic rising

--

constraints:
* certain dependencies on target environment: POSIX, ethernet
* noise by OS (scheduling!) included

--

A word on metric functions..

--

* candidates for measuring execution time: 
* cycle count <!-- .element: class="fragment" data-fragment-index="1" -->
* process/thread clock<!-- .element: class="fragment" data-fragment-index="2" -->
* (wall) clock<!-- .element: class="fragment" data-fragment-index="3" -->

--

So why choosing wall clock?
* availability of plain clock function 
* evaluation designed to mitigate noise
* framework designed to work with other clock functions

Note:
* other metric, then part of evaluation no longer necessary

--

means to mitigate noise
* explicit instrumentation of I/O syscalls
* all entities reside on different physical hardware during experiment
* core pinning to prevent context switches between CPU cores
* highest priority of all user space processes

Note:
* taskset
* nice level < 0

--

future work:
* stream ciphers
* TLS extensions (session caching)
* DTLS

--

# Questions?
