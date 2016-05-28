:warning: Currently, this repository is only documentation

---

# tcp-gate

Configuration-based TCP gateway

* Proxy any TCP listener to any TCP endpoint
* Round robin
* Caddy for TCP/TLS
* Automatic TLS via Lets Encrypt
* Management UI
* Management API
* Zero-downtime reconfigures

### Configuration specification

* Data types are defined with angle brackets `<` `foo` `>`
* Optional config are surrounded with square brackets `[bar:]bazz`

```
<tcp-gate-config> = {
  <endpoint>: <endpoint-config>
  ...
}

<endpoint> = "[<protocol>:]<abs path to unix socket>|[<host|ip>:]<port>"

<protocol> = "tcp|tls|ws|wss" (default tcp)

<endpoint-config> = {
  type: <endpoint-type>
  <<endpoint-type>-config>
}

//enum of endpoint types
<endpoint-type> = "proxy|sni|admin|loadbalance" (default proxy)

<proxy-config> = {
  remote: <endpoint>
  certificates: <certificate-config>
  timeout: {
    read: <milliseconds> (default 0)
    write: <milliseconds> (default 0)
  }
  buffer: {
    read: <bytes> (default 4096)
    write: <bytes> (default 4096)
  }
}

<certificate-config> = {
  key: <file-path>
  cert: <file-path>
}

//server-name-indication multiplexer
//missing certificates are fetched from letsencrypt
//and inserted into the appropriate <proxy-config>
<sni-config> = {
  lets-encrypt {
    agree: true|false, (default true)
    email: <email-address>
    storage: "config|file"
  }
  proxies {
    <sni host>: <endpoint-config>
    ...
  }
  certificates {
    <sni host>: <certificate-config>
    ...
  }
}

//entrypoint into the administration portal,
//an http server providing a web ui and rest api
<admin-config> = {
  auth = "<user>:<pass>"
  stats = <stats-settings>
  api = <api-settings>
}

<loadbalance-config> = {
  strategy = "round-robin|random|least-connections" (default round-robin)
  proxies [
    <endpoint-config>
  ]
}
```

**TODO: Convert to JSON Schema**

### Configuration example

```json
{
  "80": {
    "type": "proxy",
    "endpoint": "bar.company.com:9002",
    "timeout": {"read":5000, "write":5000}
  },
  "443": {
    "type": "sni",
    "proxies": {
      "web.foobar.com": {
        "type": "proxy",
        "endpoint": "bazz.company.com:9003"
      },
      "admin.foobar.com": {
        "type": "admin",
        "auth": "user:s3cret"
      }
    }
  }
}
```

### Implementation

A set of `net.Listener`s will accept a larger set of local `net.Conn`s which forward through to a set of remote `net.Conn`s.

