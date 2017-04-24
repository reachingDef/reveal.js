Outline

1. Background
2. Test bench
3. Results
4. Evaluation

--

Motivation
* IOT, Industry 4.0, cyber physical systems..
* huge number of: <!-- .element: class="fragment" data-fragment-index="1" -->
* resource constrained devices<!-- .element: class="fragment" data-fragment-index="1" -->
* connected to each other<!-- .element: class="fragment" data-fragment-index="1" -->
* real time requirements<!-- .element: class="fragment" data-fragment-index="1" -->
* How can we protect sensitive data?   <!-- .element: class="fragment" data-fragment-index="2" -->

--

Hypothesis: TLS is suited for the protection of low-resource realtime systems
* low resource = limited RAM, persistent storage and computational capacity<!-- .element: class="fragment" data-fragment-index="1" -->
* realtime = fixed (and low!) latency<!-- .element: class="fragment" data-fragment-index="2" -->

--

latency
* roughly: time between call of TLS library function until system IO function is reached
* channel establishment<!-- .element: class="fragment" data-fragment-index="1" -->
* exchange of user data  <!-- .element: class="fragment" data-fragment-index="1" -->

--

Why (not) TLS?
*  well known protocol
*  many implementations
*  considered to be slow <!-- .element: class="fragment" data-fragment-index="1" -->
*  and resource hungry <!-- .element: class="fragment" data-fragment-index="1" -->

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
* embracing log points form a log block
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

Meta vs Primitive blocks
* <b>Meta</b>: large procedure <br/>
(mbedtls_ssl_handshake) 
* <b>Primitive</b>: discrete algorithm 
(mbedtls_aes_setkey_dec) 
* relation used to judge preciseness of instrumentation (coverage) <!-- .element: class="fragment" data-fragment-index="3" -->

--

Where to place the log points?

--

What to instrument?
* cover protocol step functions
* cover the whole crypto API
* idea: most time is spent in (a)symmetric crypto operations
* knowledge of protocol step sufficient, no general need for further insight

Note:
* mbed TLS comes with a built-in benchmark tool that measures raw crypto performance
* went through list of included header files to identify roughly all crypto functions
* perks: by instrumenting the function declaration and not explicit calls, all calls are
implicitly instrumented
* crypto functions considered to be "primitive" (no need to look insight them, impl. irrelevant)

--

Software architecture

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

Procedure
1. controller assigns server / client roles <!-- .element: class="fragment" data-fragment-index="1" -->
2. for each cipher suite<!-- .element: class="fragment" data-fragment-index="2" -->
    * client initiates handshake with server
    * after channel establishment, client sends packets to server
    * instrumented data is sent to the controller
3. controller switches server / client roles<!-- .element: class="fragment" data-fragment-index="3" -->
4. repeat with 2.<!-- .element: class="fragment" data-fragment-index="4" -->

--

<img height="600"; width="auto"; src="images/ProtoInitial.png"/>

--

<img height="600"; width="auto"; src="images/ProtoNormal.png"/>

Note:
* CLI_INS_HANDSHAKE has parameter to enforce cipher suite

--

<img height="600"; width="auto"; src="images/ProtoBufferFull.png"/>

--

# Results
1. Overview
2. Toolset

--

# overview

--

Instrumented latency times (for each cipher suite)

* 1000 handshakes (initiator, responder)
* 10,000 reads / writes of 1KByte

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

Recommendation
* read/write: AES-128-CBC (AES-128-GCM if AES hardware accelerated)
* handshake: PSK (if infrastructure available), RSA
* avoid 3DES and DH(E)
* reasonable choice: TLS-RSA-WITH-AES-128-GCM

--

# The toolbox

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

.. and many more

--

# Evaluation

--

Performance impact
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

Added overhead to handshake latency (Board perspective)
<img src="images/global_difference_write_latencies_by_keyxchg_PC_AS_SERVER.svg"/>
Note:
* instrumentation in driver
* time to transmit logs captured in both cases
* y-axis: relation of median handshake latency of 1000 runs between instrumented/uninstrumented mbed TLS 
* x-axis: relation of median user data exchange latency of 10000 runs between instrumented/uninstrumented mbed TLS
* also colored by symmetric
* PSK most affected, DHE not (fits to the other results)

--

Added overhead to user data exchange latency (PC perspective)
<img src="images/global_difference_read_latencies_by_sym_PC_AS_SERVER.svg"/>
Note:
* average of all data points is slightly below 1
* noise indicates that logging has hardly impact on fast machines 

--

Requirements / Constraints:
* certain dependencies on target environment: POSIX, ethernet
* noise by OS (scheduling!) included

--

A word on metric functions..

--

candidates for measuring execution time
* cycle count <!-- .element: class="fragment" data-fragment-index="1" -->
* process/thread clock<!-- .element: class="fragment" data-fragment-index="2" -->
* (wall) clock<!-- .element: class="fragment" data-fragment-index="3" -->

Note:
* RDTSC (read time stamp)

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
* porting to actual embedded platforms
* stream ciphers 
* TLS extensions 
* DTLS
* further variation of parameters

Note:
* ChaCha20, Poly1305 
* session caching
* UDP
* different packet sizes

--

# Questions?
