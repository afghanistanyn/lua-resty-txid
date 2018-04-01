# lua-resty-txid

[![CircleCI](https://circleci.com/gh/GUI/lua-resty-txid.svg?style=svg)](https://circleci.com/gh/GUI/lua-resty-txid)

lua-resty-txid provides a function that can be used to generate unique transaction/request IDs for [OpenResty/nginx](http://openresty.org). The IDs can be used to correlate logs or upstream requests and have the following characteristics:

- 20 characters
- base32hex encoded
- Temporally and lexically sortable
- Case insensitive
- 96 bit identifier

lua-resty-txid is a LuaJIT port of [ngx\_txid](https://github.com/streadway/ngx_txid) for [OpenResty](https://openresty.org/) (or nginx with [ngx\_lua](https://github.com/openresty/lua-nginx-module#installation)). The IDs generated by lua-resty-txid follow the exact same pattern and are compatible with ngx\_txid.

## Installation

Via OPM:

```sh
opm get GUI/lua-resty-txid
```

Or via Luarocks:

```sh
luarocks install resty-txid
```

## Usage

A single `txid()` Lua function is exposed by this module to generate IDs:

```lua
local txid = require "resty.txid"
local id = txid() -- b2g6q94qdn6h84an7vfg
```

Each time `txid()` is called, a new, unique ID will be returned, so you will need to cache the result if you wish to reuse the same ID in multiple places for a single request. Depending on your usage, [`ngx.ctx`](https://github.com/openresty/lua-nginx-module#ngxctx) or [`set_by_lua`](https://github.com/openresty/lua-nginx-module#set_by_lua) offer some simple options for caching the value on a per-request basis.

```lua
txid() -- b2g83t2oshrg092mjggg
txid() -- b2g83t2oodncokuges00

ngx.ctx.txid = txid() -- b2g83t2od939mdvb2l0g
ngx.ctx.txid          -- b2g83t2od939mdvb2l0g
```

Finally, `txid()` accepts an optional argument for what timestamp (in milliseconds) to use when generating the ID. By default, the current timestamp is used. Since the resulting IDs are temporally and lexically sortable, this can be used to generate IDs that will be sorted based on a previous date or time.

```lua
local timestamp_ms = 655829050000 -- 1990-10-13 14:44:10
txid(timestamp_ms) -- 4om9qi54la8ffr4bd9sg

local timestamp_ms = 655929050000 -- 1990-10-14 12:30:50
txid(timestamp_ms) -- 4on1lg74nt0ud2ssllu0
```

### Example

A more complete example, with caching, setting request/response headers, and integration with nginx's logging:

```nginx
http {
  log_format agent "$lua_txid $http_user_agent";
  log_format addr "$lua_txid $remote_addr";

  init_by_lua_block {
    # Pre-load the module.
    require "resty.txid"
  }

  server {
    listen 8080;
    access_log logs/agents.log agent;
    access_log logs/addrs.log addr;

    # Set an nginx variable that is cached per request and can be used in the
    # nginx log_format.
    set_by_lua_block $lua_txid {
      local txid = require "resty.txid"
      return txid()
    }

    location / {
      # Set a header on the response providing the ID.
      more_set_headers "X-Request-Id: $lua_txid";

      # Set a header on the request providing the ID (which will be sent to the
      # proxied upstream).
      more_set_input_headers "X-Request-Id: $lua_txid";

      proxy_pass http://localhost:8081;
    }
  }
}
```

## Performance

Benchmarks indicate that performance is equivalent to the [ngx\_txid](https://github.com/streadway/ngx_txid) C extension.

## Design

The transaction ID design is a direct port of [ngx\_txid](https://github.com/streadway/ngx_txid), so here's all the original information about the design from ngx\_txid:

### Background

The design of this transaction ID should meet the following requirements:

- Be roughly numerically temporally sortable with ~second granularity.
- Have a representation that is roughly lexically sortable with ~second granularity.
- Have a probability of less than 1e-9 for collision at 1 million transactions per second.
- Be efficient and easy to decode into fixed size C types
- Always be available at the risk of higher collision probability
- Use as few bytes as possible
- Work with IPv4 and IPv6 networks

### Technique

Use a monotonic millisecond resolution clock in the high 42 bits and system entropy for the low 54 bits. Use enough entropy bits to satisfy a collision probability at a desired global request rate.

```
+------------- 64 bits------------+--- 32 bits ----+
+------ 42 bits ------+--22 bits--|----------------+
| msec since 1970-1-1 | random    | random         |
+---------------------+-----------+----------------+
```

A request rate of 1 million per second across all servers means 1000 random values per millisecond.  Estimating the collision probability using the [birthday paradox](http://en.wikipedia.org/wiki/Birthday_problem) can be done with this formula: `1 - e^(-((m^2)/(2*n)))` where `m` is the number of ids and `n` is the number of random values possible.

When using 54 bits of entropy:

```
1mil req/s  = 1 - exp(-((1000^2) /(2*2^54))) = 2.775558e-11
10mil req/s = 1 - exp(-((10000^2)/(2*2^54))) = 2.775558e-09
```

The odds of collision are small even at 10 million requests per second.

Nginx keeps track of the current clock in increments of the configuration directive `timer_resolution`.  The clock resolution for `$txid` is 1ms, so a timer resolution greater than 1ms means that the probability of collision will increase.  If you have a `timer_resolution` of 10ms, 1 million requests per second would require 10,000 random values per second in the worst case.

### Encoding

[base32hex](https://en.wikipedia.org/wiki/Base32#base32hex) is used with a lower case alphabet and without padding characters is chosen for the following reasons:

- Lexically sort order equivalent to numeric sort order
- Case insensitive equality
- Lower case is easer for visual compares
- Denser than hex encoding by 4 bytes

### Other techniques

- [snowflake](https://github.com/twitter/snowflake): Uses time(41) + unique id(10) + sequence(12).
  - Pro: Guaranteed unique sequences
  - Pro: Fits in 63 bits
  - Cons: Requires unique id coordination for each server - 16 workers processes per host means a limit of 64 instances of nginx
  - Cons: Only 11 bits available for unique id, needs monitoring
  - Cons: Total ordering only possible in the same process
  - Cons: Service interruption possible when clocks lose synchronization

- [flake](https://github.com/boundary/flake): Uses time + mac id + sequence.
  - Pro: Guaranteed unique sequences
  - Cons: Uses 128 bits
  - Cons: Wastes 22 bits of timestamp data
  - Cons: Only a single process per host can generate ids - needs to synchronize access to the sequence from each worker process
  - Cons: Service interruption possible when clocks lose synchronization
  - Cons: Seeds cross platform MAC Address lookup.

- [UUIDv4](http://www.ietf.org/rfc/rfc4122.txt): 122 bits of entropy
  - Pro: Very low probability of collision
  - Cons: Unsortable

- [UUID with timestamp](http://www.ietf.org/rfc/rfc4122.txt): 48 bits of time + 74 bits entropy
  - Pro: Very low probability of collision
  - Cons: String representation is not temporally local

- [httpd mod\_unique\_id](http://httpd.apache.org/docs/2.4/mod/mod_unique_id.html): Host ip(32) + pid(32) + time(32) + sequence (16) + thread id (32)
  - Pro: Deterministic
  - Cons: Uses 144 bits
  - Cons: Assumes unique IPv4 for the hostnamme's interface
  - Cons: Unsortable case-sensitive custom representation - base64 with a custom alphabet
  - Cons: Hard limit of 65535 ids per second per pid - small tolerance for clock steps

## Development

After checking out the repo, Docker can be used to run the test suite:

```sh
docker-compose run --rm app make test
```

## Credits

Credit for the original design goes to [ngx\_txid](https://github.com/streadway/ngx_txid). This is simply a pure LuaJIT port for easier installation in OpenResty.