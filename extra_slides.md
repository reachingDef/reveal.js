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

Logging labels
* translated before compile time 
* combination of identifier and type
* type can either be primitive or compound ("meta")
* example: 
        PERFORM_BENCHMARK,META
becomes

        enum logging_label {
            PERFORM_BENCHMARK_START = 0;
            PERFORM_BENCHMARK_STOP = 1;
        }

Note:
* typically with _START and _STOP suffixes
* depending on identifier, actual labels are generated
* additional features possible (e.g. for use during analysis)
* Data structures
* logging_context
* logging_entry
* payload
* field for additional information
* could be used for arbitrary types or information
* example: energy consumption value

--

