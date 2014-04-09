libasyncd
=========

Embeddable Event-based Asynchronous Message/HTTP Server library in C.

## What is libasyncd?

Libasyncd is an embeddable event-driven asynchronous message server for C/C++.
It supports HTTP protocol by default and you can add your own protocol handler(hook)
to build your own high performance server.

Asynchronous way of programming can easily go quite complicated since you need to
handle every possible things in nonblocking way. So the goal of Libasyncd project is
to make a flexible and fast asynchronous server framwork with nice abstraction that
can cut down the complexity.

## Why libasyncd?

* Stands as a generic event-based server library.
* Not only for HTTP server but also as a RPC server, as a Protocol Buffer channel,
  as a Message transforming layer...
* Embeddable library module - you write main().
* Simple to use.
* Pluggable protocols.
* HTTP protocol handler (support chunked transfer-encoding)
* Support of multiple hooks.
* Support request pipelining.
* Support SSL - Just flip the switch on.

## Compile & Install.
```
$ git clone git clone https://github.com/wolkykim/libasyncd
$ cd libasyncd
$ (cd lib; run2init-submodules.sh; cd qlibc; ./configure; make) 
$ ./configure
$ make
```

## "Hello World", Asynchronous Socket Server example.
```
int my_bypass_handler(short event, ad_conn_t *conn, void *userdata) {
    evbuffer_add(conn->out, "Hello World.");
    return AD_CLOSE;
}

int main(int argc, char **argv) {
    ad_server_t *server = ad_server_new();
    ad_server_set_option(server, "server.port", "2222");
    ad_server_register_hook(server, my_bypass_handler, NULL);
    return ad_server_start(server);
}
```

## "Hello World", Asynchronous HTTP Server example.
```
int my_bypass_handler(short event, ad_conn_t *conn, void *userdata) {
    if (ad_http_get_status(conn) == AD_HTTP_REQ_DONE) {
        ad_http_response(conn, 200, "text/html", "Hello World", 11);
        return AD_DONE;
    }
    return AD_OK;
}

int main(int argc, char **argv) {
    ad_server_t *server = ad_server_new();
    ad_server_set_option(server, "server.port", "8080");
    ad_server_register_hook(server, ad_http_handler, NULL); // HTTP Parser is also a hook.
    ad_server_register_hook_on_method(server, "GET", my_bypass_handler, NULL); // Put yours after parser.
    return ad_server_start(server);
}
```

## Looking for more controls? Echo server example here.

```
/**
 * User data for per-connection custom information for non-blocking operation.
 */
struct my_cdata {
    int counter;
};

/**
 * This callback will be called before closing or resetting connection for
 * pipelining.
 */ 
void my_userdata_free_cb(ad_conn_t *conn, void *userdata) {
    free(userdata);
}

/**
 * User callback example.
 *
 * This is a simple echo handler.
 * It response on input line up to 3 times then close connection.
 *
 * @param event event type. see ad_server.h for details.
 * @param conn  connection object. type is vary based on
 *              "server.protocol_handler" option.
 * @userdata    given shared user-data.
 *
 * @return one of AD_OK | AD_DONE | AD_CLOSE | AD_TAKEOVER
 *
 * @note Please refer ad_server.h for more details.
 */
int my_conn_handler(short event, ad_conn_t *conn, void *userdata) {
    DEBUG("my_conn_callback: %x", event);

    /*
     * AD_EVENT_INIT event is like a constructor method.
     * It happens only once at the beginning of connection.
     * This is a good place to create a per-connection base
     * resources. You can attach it into this connection to
     * use at the next callback cycle.
     */
    if (event & AD_EVENT_INIT) {
        DEBUG("==> AD_EVENT_READ");
        // Allocate a counter container for this connection.
        struct my_cdata *cdata = (struct my_cdata *)calloc(1, sizeof(struct my_cdata));

        // Attach to this connection.
        ad_conn_set_userdata(conn, cdata, my_userdata_free_cb);
    }

    /*
     * AD_EVENT_READ event happens whenever data comes in.
     */
    else if (event & AD_EVENT_READ) {
        DEBUG("==> AD_EVENT_READ");

        // Get my per-connection data.
        struct my_cdata *cdata = (struct my_cdata *)ad_conn_get_userdata(conn);

        // Try to read one line.
        char *data = evbuffer_readln(conn->in, NULL,  EVBUFFER_EOL_ANY);
        if (data) {
            if (!strcmp(data, "SHUTDOWN")) {
                //return AD_SHUTDOWN;
            }
            cdata->counter++;
            evbuffer_add_printf(conn->out, "%s, counter:%d, userdata:%s\n", data, cdata->counter, (char*)userdata);
            free(data);
        }

        // Close connection after 3 echos.
        return (cdata->counter < 3) ? AD_OK : AD_CLOSE;
    }

    /*
     * AD_EVENT_WRITE event happens whenever out-buffer has lesser than certain
     * amount of data.
     *
     * Default watermark is 0 meaning this will happens when out-buffer is empty.
     * For reasonable size of message, you can send it all at once but for a large
     * amount of data, you need to send it out through out multiple callbacks.
     *
     * To maximize the performance, you will also want to set higher watermark
     * so whenever the level goes below the watermark you will be called for the
     * refill work before the buffer gets empty. So it's just faster if you can
     * fill up gas while you're driving without a stop.
     */
    else if (event & AD_EVENT_WRITE) {
        DEBUG("==> AD_EVENT_WRITE");
        // We've sent all the data in out-buffer.
    }

    /*
     * AD_EVENT_CLOSE event happens right before closing connection.
     * This will be the last callback for this connection.
     * So if you have created per-connection data then release the resource.
     */
    else if (event & AD_EVENT_CLOSE) {
        DEBUG("==> AD_EVENT_CLOSE=%x (TIMEOUT=%d, SHUTDOWN=%d)",
              event, event & AD_EVENT_TIMEOUT, event & AD_EVENT_SHUTDOWN);
        // You can release your user data explicitly here if you haven't
        // set the callback that release your user data.
    }

    // Return AD_OK will let the hook loop to continue.
    return AD_OK;
}

int main(int argc, char **argv) {
    // Example shared user data.
    char *userdata = "SHARED-USERDATA";

    //
    // Create a server.
    //
    ad_server_t *server = ad_server_new();

    //
    // Set server options.
    //
    // Usually you only need to override a few default options and
    // that's it but it's lengthy here for a demonstration purpose.
    //
    ad_server_set_option(server, "server.port", "2222");
    ad_server_set_option(server, "server.addr", "0.0.0.0");
    ad_server_set_option(server, "server.timeout", "5");

    // Set protocol handler.
    //   - bypass : Use bypass handler. This is a transparent handler for
    //              build an custom protocols.
    //   - http   : Use HTTP handler. Request message will be parsed by
    //              the handler. You can put your hooks on method name or
    //              on each phase of parsing process like "AFTER_HEADER".
    //   - euca   : Use EUCA handler. This handler is for EUCA message.
    //              light weight messaging protocol designed for the
    //              blasting fast performance in data exchange.
    ad_server_set_option(server, "server.protocol_handler", "http");

    // Register custom hooks. When there are multiple hooks, it will be
    // executed in the same order as it registered.
    ad_server_register_hook(server, my_conn_handler, userdata);

    // SSL options.
    ad_server_set_option(server, "server.server.enable_ssl", "0");
    ad_server_set_option(server, "server.ssl_cert", "/usr/local/etc/ad_server/ad_server.cert");
    ad_server_set_option(server, "server.ssl_pkey", "/usr/local/etc/ad_server/ad_server.pkey");

    // Enable request pipelining, this change AD_DONE's behavior.
    ad_server_set_option(server, "server.request_pipelining", "1");

    // Run server in a separate thread. If you want to run multiple
    // server instances or if you want to run it in background, this
    // is the option.
    ad_server_set_option(server, "server.daemon", "0");

    // Call ad_server_free() internally when server is shutting down.
    ad_server_set_option(server, "server.free_on_stop", "1");

    //
    // Start server.
    //
    int retstatus = ad_server_start(server);

    //
    // That is it!!!
    //
    return retstatus;
}
```

## References

* [C10K problem](http://en.wikipedia.org/wiki/C10k_problem)
* [libevent library - an event notification library](http://libevent.org/)
* [qLibc library - a STL like C library](http://wolkykim.github.io/qlibc/)
