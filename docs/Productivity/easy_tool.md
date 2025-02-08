# 简单小工具合集

## go-wrk http压测神器

* <https://github.com/tsliwowicz/go-wrk>

```text
   Usage: go-wrk <options> <url>
   Options:
    -H       Header to add to each request (you can define multiple -H flags) (Default )
    -M       HTTP method (Default GET)
    -T       Socket/request timeout in ms (Default 1000)
    -body    request body string or @filename (Default )
    -c       Number of goroutines to use (concurrent connections) (Default 10)
    -ca      CA file to verify peer against (SSL/TLS) (Default )
    -cert    CA certificate file to verify peer against (SSL/TLS) (Default )
    -d       Duration of test in seconds (Default 10)
    -f       Playback file name (Default <empty>)
    -help    Print help (Default false)
    -host    Host Header (Default )
    -http    Use HTTP/2 (Default true)
    -key     Private key file name (SSL/TLS (Default )
    -no-c    Disable Compression - Prevents sending the "Accept-Encoding: gzip" header (Default false)
    -no-ka   Disable KeepAlive - prevents re-use of TCP connections between different HTTP requests (Default false)
    -no-vr   Skip verifying SSL certificate of the server (Default false)
    -redir   Allow Redirects (Default false)
    -v       Print version details (Default false)
```

example

```bash
go install github.com/tsliwowicz/go-wrk@latest

## 一个测试案例
go-wrk -c 80 -d 5 -M="POST" -H "Name1: Value1" -H "Name2: Value2" -H "Name3: Value3" -body '{"accountID":"1"}' http://localhost:1234/geInfo
```