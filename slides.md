## TLS for protecting resource-constrained systems
Yannic Ahrens

--

Outline

--

Hypothesis: TLS is suited for the protection of resource-constrained systems
Aufbau: mit IOT bullshit motivieren, dann grob die Methodik der Arbeit skizzieren (latency, constrained),
anschließend schritt für schritt ins detail gehen

--

metrics (1) 10.000m view:
* real time = fixed (and low!) latency
* resource constrained = limited RAM and persistent storage

--

metrics (2) 10.000m view:
we can derive the two objectives:
* RT: minimize additional time when TLS is involved in the communication (security??)
* contrained: needs to run on target platform (constrained by hardware and OS environment)

--

relevant aspects from user perspective
* handshake
* payload exchange

--

Verification in two steps:
1. development of a test bench
2. evaluation of TLS on selected target platforms

--

Background:
* TLS works upon transport layer
* relevant sub protocols: Handshake & record layer protocol

--

client initiates connection through ClientHello, sends list of supported cipher suites
server picks one cipher suite, responds
authentication through certificate or pre-shared secret
depending on authentication method, key exchange method varies

--

Cipher suites are a fixed combination of mechanisms:
- authentication
- key exchange
- hashing
- bulk cipher

--

Why is choice of cipher suite so important?
* one of the very few dynamic components
* cryptographic operations dominate the runtime

--

What did I do?
* picked a TLS library and instrumented the implementation
* use a very simple experiment setup
* test all cipher suites for their influence on latency

--

candidate for TLS library
* OpenSSL (and derivates): too heavy
* wolfSSL: suited for embedded environments, seems somewhat "hacky"
* mbed TLS: ARMs official mbed library, supports almost all TLS features, felt convenient to use

--

structure of mbed
* split into three libraries: crypto, tls, x509
* gained insight into library via callgrind (and KCachegrind)
* TLS protocols steps each have their own function 
* write_xy: caller sends the message xy, parse_xy: caller receives the message xy

--

How to instrument?
* callgrind: excellent tool, just too heavy
* gprof: had a lot of quirks, also not sure if available on target platforms
* considered other papers and their approach, none really suited mine
* black box approach: device in the middle ("proxy"), does not allow deep insight

--

Instrumentation
* small library linked to target code
* manual insertion of log points into the code
* relies on clock gettime call
* different clock modes (wall clock, NTP independent clock, process / thread time)
* notifies controller if internal buffer is full and logs need to be fetched
* two important structures: logging_context, logging_entry

--

API
* log_point(label, ctx, result)
Usage
    int my_function(...) {
        log_point(MY_FUNCTION_START, global_log_ctx, 0);
        // do stuff
        log_point(MY_FUNCTION_STOP, global_log_ctx, 0);
        return result;
    }

--

Logging labels
* identifier and type
* depending on identifier, actual labels are generated
* typically with _START and _STOP suffixes
* additional features possible (e.g. for use during analysis)
* generated before compile time, put into a header file

--

Log trees
* nested log points form a log tree
* <insert graphic!>
* similar to matching many different parenthesises

--

Procedure (Instrumentation)
* setup network
* run start_bench.sh on target platforms
* start_bench.sh gets server IP, server TLS port and a port on which the controller is accepted
* controller.py knows addresses through config file
* controller.py then told which experiment to run, initiates experiments
* bench processes run experiments (repeatedly!); are eventually queried by controller.py
* controller stores logs into database for further analysis

--

Procedure (Analysis)
* two phases: extraction of useful information (put into csvs) and plotting of the data
* reason: interpretation of logs time consuming, intermediate dumps speed workflow up

--

log storage
* SQLite (very homogenous data)
* split into multiple database files (performance)
* directory structure: <exp-ID>/<exp-type>/<Board or pc as server>/<cs-id>/<Pc or board>.ins
* ins for instrumentation

--

* dump list of all supported cipher suites by this built of mbed into file 
* reasons: explicit knowledge on current CS, builts for very constrained platforms may be stripped
* would have to do several builds and runs to test all CS

--

builder.py
* generates header for configuration and log labels
* builds mbed TLS and bench

--

placement of logs
* cover protocol step functions
* cover the whole crypto API
* idea: most time is spent in (asymmetric crypto operations)
* knowledge of protocol step sufficient, no general need for further insight

--

covering the crypto API
* mbed TLS comes with a built-in benchmark tool that measures raw crypto performance
* went through list of included header files to identify roughly all crypto functions
* perks: by instrumenting the function declaration and not explicit calls, all calls are
implicitly instrumented
* crypto functions considered to be "primitive" (no need to look insight them, impl. irrelevant)

--

completeness (1)
* "How to ensure I haven't forgotten to instrument a major spot?"
* remember log trees: log points form a block through _START and _STOP labels
* by instrumenting top-level "do_experiment_function", all execution time is taken into account
* from there, things get only more precise

--

completeness (2)
* a high level log block is made of primitive log blocks and time that's unaccounted for
* ratio between unaccounted time and time of primitive log blocks determine the coverage or insight into the target
* hence all time is measured. constraints: possibly I cannot figure the exact spot yet. can be fixed by adding more log points
* for this to work, target must be single-threaded (mbed TLS is)

--

experiment (1) (additional graphics)
* three parties: controller, TLS server and client
* controller loads list of all cipher suites
* controller sends current CS to client
* (number of iterations, total payload size, size of a single packet)
* here (1000, 10000, 1000)
* read: each cipher suite was measured 1000 times. During each iteration,
a total of 10000 bytes was exchanged, where each packet was 1000 byte large
* since TLS asymmetric protocol: roles change

--

Internally
* client initiates and proceeds handshake 10 times
* after each channel establishment, 10x 1KB dummy payload is sent from client to server
* repeat after all cipher suites were used
* then start again (total: 100 iterations)
* reason: whole experiment for single CS takes very long. By splitting up,
already measured logs can be processed. Also, an "open end" scenario is thinkable

--

performance impact
* measured on two levels
* micro: run two crypto functions (slow, fast) with different forms of logging
* different forms = no logging, logging with metric function, two different clock modes
* macro: compare the runtime of an instrumented version of mbed TLS with an instrumented

--

micro

--

macro
* instrumentation still takes place in bench process
* no deeper insight possible (IO ?)
* compare handshake and read and write latency

--

overhead, pc as server, pc side
<img src="images/global_difference_read_latencies_by_keyxchg_PC_AS_SERVER.svg"/>

--

overhead, pc as server, board side
<img src="images/global_difference_write_latencies_by_keyxchg_PC_AS_SERVER.svg"/>

--

overhead sym, pc as server, pc side
<img src="images/global_difference_read_latencies_by_sym_PC_AS_SERVER.svg"/>

--

overhead sym, pc as server, board side
<img src="images/global_difference_write_latencies_by_sym_PC_AS_SERVER.svg"/>

--

results
* server vs. client (handshake, payload)
* handshake vs payload
* type ratio
* hash vs sym
* stacked bar handshake proto steps
* density & histograms
* correlation time and packet no.

--

verification
* measure overhead (micro & macro scale)
* dependency of proto steps on cipher suite
* argumentative
* unaccounted areas

--

constraints:
* certain dependencies on target environment
* noise by OS (scheduling!) included

--

future work:
* stream ciphers
* TLS extensions (session caching)
* DTLS

--

portability and flexibility:
* requirements on metric function: monotonic rising
* POSIX
* other clock functions (process time) or even metrics (energy possible)

--

