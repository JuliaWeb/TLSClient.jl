# TLSClient.jl


WIP/Proposal for a Julia client interface for OS-native TLS over TCP.


## Requirements

**Small API**

The API has the minimum number of functions and options required to
abstract the underlying implementations and to support HTTPS. 

_Rationale:_
 - _Minimise effort required to add a new implementations_. 
 - _Avoid 2nd class emulations of special features that are only
   available on some platforms. Special features can go in a
   separate API. HTTPS is the most common use-case._


**TCP Connection hidden by API**

The API does not expose the underlying TCP connection or raw file descriptors.

_Rationale:_
 -  _Makes the Small API requirement easier to meet._
 -  _Allows use of platform APIs where the interaction between the
    encryption layer and TCP/IP stack is integrated/optimised, or_
 -  _where the TCP layer is not accessible (e.g embedded system with
    external WiFI module)._


**Non blockig API**

The API calls are all non-blocking (except for `wait(::TLSStream)`).
This includes no waiting for the network and no waiting for
[internal locks](https://github.com/JuliaWeb/MbedTLS.jl/blob/master/src/ssl.jl#L211).

_Rationale:_
 - _Makes the Small API requirement easier to meet._
 - _Reduces the chance of user visible platform behaviour differences
   in timing and sequencing (which can result in race conditions,
   deadlocks etc)._
 - _Blocking APIs can be implemented  using `wait` at a higher layer._


**Common wait implementation**

`wait(tls::TLSStream)` should just call `poll_fd` on a `RawFD` (in cases
where the platform implementation has a `RawFD` for the connection).

_Rationale:_
 - _Try to avoid behaviour differences tha might arise from different
   ways of polling and pumping the Julia event loop._
 - _Take advantage of future improvements in `poll_fd`._


**C Abstraction layer**

The Julia API interfaces with a single common C header using simple C types.
Each platform implementation provides a dynamic library that implements this
C interface.

_Rationale:_
 - _Avoid fragile mapping of C structs into Julia code._
 - _Support platforms where the native API is not compatible with
   `ccall` (C++?)._
 - _Allow more direct use of platform refernece examples. e.g.
   "Here is how you connect a Schannel to a WinSock...",
   less risk of introducing bugs if that is not all translated into `ccalls`._
 - _Simplify creation of minimal breaking examples when submitting bug reports
   to vendors_.
 - _Make things easier for external security auditors to understand._
            

## API Sketch

`tls_client.h`:

```C
// All functions returns 1 on success, 0 on failure.
// On failutre: *err is an error code , e.g. :TLS_LIBRARY_NOT_FOUND,
// and *errmsg is a description of what went wrong.


// One time global library initilisation.
int tls_init(jl_sym_t** err, char** errmsg);


// Connect to TCP host:port and start TLS handshake.
// tlsout returns a connection handle.
// Note: non-blocking, so any connection or handshake errors won't be
// reported until one of the other API functions is called on this handle.
int tls_connect(char* host, char* port,
                void* tlsout,
                jl_sym_t** err, char** errmsg);


// Close the conncetion.
int tls_close(void* tls, char** err, char** errmsg)


// Is the connection open?.
// isopen returns 1 or 0.
int tls_close(void* tls, int* isopen, char** err, char** errmsg)


// Send bytes.
// nout returns number of bytes sent from buf.
int tls_write(void* tls, uint8_t* buf, size_t n, size_t* nout,
              jl_sym_t** err, char** errmsg);


// How many bytes are available to read?
// nbytes returns number of bytes available to read.
int tls_bytesavailable(void* tls, int* nbytes,
                       jl_sym_t** err, char** errmsg);


// Recieve bytes.
// nin returns number of bytes read into buf.
int tls_read(void* tls, uint8_t* buf, size_t n, size_t* nin,
             jl_sym_t** err, char** errmsg);


// Wait (for connection, handshake, read/write network activity etc).
int tls_wait(void* tls, int timeout_ms)
             jl_sym_t** err, char** errmsg);
```


## Implementations

List of Platform Vendor Supported TLS implementations:
 - MS Windows: [MS SSPI Schannel](https://msdn.microsoft.com/en-us/library/windows/desktop/aa374782(v=vs.85).aspx)
 - iOS/macOS: [Apple Security.SecureTransport](https://developer.apple.com/documentation/security/secure_transport)
 - AWS: [Amazon s2n](https://github.com/awslabs/s2n)
 - Linux: [OpenSSL](https://www.openssl.org)
 - Mbed OS: [ARM Mbed TLS](https://github.com/ARMmbed/mbedtls)
