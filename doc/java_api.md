# DDlog Java API

This document describes the steps required to invoke a DDlog program from Java,
based on the `test/datalog_tests/redist.dl` example DDlog program and Java code
in `java/test_flatbuf`.

## Prerequisites

See the [README](https://github.com/vmware/differential-datalog#prerequisites)
for instructions on how to install the Google FlatBuffers library required to 
compile DDlog Java bindings.

## Compile DDlog Java bindings

Skip this step if you are using a binary release of DDlog, where pre-compiled
Java bindings can be found in `java/ddlogapi.jar`.  To compile this them from
source:

```
cd java
make
```

## Compiling DDlog programs with Java API enabled

Pass the `-j` switch to the DDlog compiler to generate Rust and Java code
required to communicate with the DDlog program from Java.  When compiling the
resulting Java program, enable the `flatbuf` feature:

```
ddlog -i test/datalog_tests/redist.dl -L lib -j
cd test/datalog_tests/redist_ddlog
cargo build --features=flatbuf --release
```

Link the compiled DDlog program along with the DDlog Java API bindings in a
single dynamic library.  Assuming the `$DDLOG` environment variable points to
the directory where DDlog is installed:

```
cc -shared -fPIC -I${JAVA_HOME}/include -I${JAVA_HOME}/include/${JDK_OS} -I. -I${DDLOG}/lib ${DDLOG}/java/ddlogapi.c -Ltarget/release/ -lredist_ddlog -o libddlogapi.so
```

(on a Mac, use the `.dylib` extension instead of `.so`).

## Linking against DDlog API

Finally, add the following dependencies to your Java project to be able to use
the DDlog API:

- The generic DDlog Java API shared by all DDlog programs in the `ddlogapi.jar`
  package (see above).

- The FlatBuffers Java runtime library, found in the `java` directory of the
  FlatBuffers distro.

- The auto-generated `ddlog` package that contains program-specific bindings
  needed to serialize and de-serialize data exchanged by Java and DDlog.  It is
  found in he `flatbuf/java` directory inside the generated
  `<progname>_ddlog` source tree.

## Using the DDlog API from Java

Here is a minimal Java program, copied from `java/test_flatbuf/Test.java` that
instantiates and uses the DDlog API.

```
import java.io.IOException;
import java.util.*;
import java.lang.RuntimeException;

/* Generic DDlog API shared by all programs. */
import ddlogapi.DDlogAPI;
import ddlogapi.DDlogCommand;

/* Additional program-specific bindings generated by `ddlog`. */
import ddlog.redist.*;

public class Test {
    private final DDlogAPI api;

    Test() {
        /* Create an instance of the DDlog program with one worker thread. */
        this.api = new DDlogAPI(1, null, false);
    }

    void onCommit(DDlogCommand command) {
        System.out.println(command.toString());
    }

    void run() {

        /* Start transaction.  All DDlog table updates must be made in the
         * context of a transaction. */
        this.api.start();

        /* Create a builder object that will be used to serialize DDlog commands
         * into a buffer. */
        redistUpdateBuilder builder = new redistUpdateBuilder();

        /* Create several DDlog commands.  Commands are stored inside the
         * builder. */
        builder.insert_DdlogNode(10000);
        builder.insert_DdlogBinding((short)100, 10000);
        builder.insert_DdlogDependency(10000, 20000);

        /* Apply commands serialized by the builder to the DDlog program. */
        int res = builder.applyUpdates(this.api);

        /* Commit transaction, triggering the `onCommit` callback for every
         * record in an output relation modified by the transaction. */
        this.api.commit_dump_changes(r -> this.onCommit(r));

        /* Terminate the DDlog program. */
        this.api.stop();
    }

    public static void main(String[] args) throws IOException {
        Test test = new Test();
        test.run();
    }
}
```

To run the program:
```
java -Djava.library.path=. Test > test.dump
```

Note the use of the `-D` switch to tell Java where to look for the `libddlogapi.so`
dynamic library.
