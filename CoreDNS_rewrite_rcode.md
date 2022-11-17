# CoreDNS Rewrite Response RCODE

## How-to

Using the [template](https://coredns.io/plugins/template/) plugin can manipulate DNS response RCODE. The following example is to make sure any DNS query would return *SERVFAIL* RCODE.

```
.:5300 {
    template ANY ANY {
        rcode SERVFAIL
    }
}
```

## Example

```
$ dig www.pinterest.com -p 5300 @localhost +nocmd +nostats
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 23848
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: c08b746c2e412c4c (echoed)
;; QUESTION SECTION:
;www.pinterest.com.		IN	A
```
