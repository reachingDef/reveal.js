## TLS for protecting resource-constrained systems
Yannic Ahrens

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

Ciphersuites are a fixed combination of mechanisms:
- authentication
- key exchange
- hashing
- bulk cipher

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

