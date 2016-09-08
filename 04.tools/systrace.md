#Use systrace for performance track

##native trace

    #define ATRACE_TAG ATRACE_TAG_INPUT

    #native
    ATRACE_CALL();

##Java trace

  #JAVA
  public void method() {
      Trace.beginSection("method");
      try {
          Trace.beginSection("section name");
          try {
              // code for Jane task...
          } finally {
              Trace.endSection(); // ends "section name"
          }
          Trace.beginSection("section name 2");
          try {
              // code for John task...
          } finally {
              Trace.endSection(); // ends "section name2"
          }
      } finally {
          Trace.endSection(); // ends "method"
      }
  }


When using application-level tracing with Systrace, you must specify the package name of your application in the user interface or specify the -a or --app= options on the command line.


##collect result

  python systrace.py --time=5 sched gfx view wm input freq -o trace001.html -a com.android.launcher3



##tags

/* Tracing categories */
static const TracingCategory k_categories[] = {
    { "gfx",        "Graphics",         ATRACE_TAG_GRAPHICS, { } },
    { "input",      "Input",            ATRACE_TAG_INPUT, { } },
