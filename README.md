# misp-guard
`misp-guard` is a [mitmproxy](https://mitmproxy.org/) addon that inspects the events that MISP is attempting to syncronize with external MISP instances via `PUSH` or `PULL` and applies a set of customizable rules defined in a JSON file.

> **NOTE: By default this addon will block all outgoing HTTP requests that are not required during a MISP server sync.**

## PUSH
```mermaid
sequenceDiagram
    participant External MISP
    participant MISP Guard
    participant Internal MISP

    Internal MISP->>MISP Guard: [GET]/servers/getVersion
    MISP Guard->>External MISP: [GET]/servers/getVersion
    External MISP->>MISP Guard: [GET]/servers/getVersion
    MISP Guard->>Internal MISP: [GET]/servers/getVersion
    
    Internal MISP->>MISP Guard: [HEAD]/events/view/[UUID]
    note right of MISP Guard: Only `minimal` search requests to /events/index are allowed
    MISP Guard->>External MISP: [HEAD]/events/view/[UUID]
    External MISP->>MISP Guard: [HEAD]/events/view/[UUID]
    MISP Guard->>Internal MISP: [HEAD]/events/view/[UUID]
    
    rect rgb(191, 223, 255)
    note left of MISP Guard: 404: If the event does not exists in External MISP
    Internal MISP->>+MISP Guard: [POST]/events/add
    note right of MISP Guard: Outgoing Event is inspected and rejected with 403 if any block rule matches
    MISP Guard->>-External MISP: [POST]/events/add
    External MISP->>MISP Guard: [POST]/events/add
    MISP Guard->>Internal MISP: [POST]/events/add
    end

    rect rgb(191, 223, 255)
    note left of MISP Guard: 200: If the event already exists in External MISP
    Internal MISP->>+MISP Guard: [POST]/events/edit/[UUID]
    note right of MISP Guard: Outgoing Event is inspected and rejected with 403 if any block rule matches
    MISP Guard->>-External MISP: [POST]/events/edit/[UUID]
    External MISP->>MISP Guard: [POST]/events/edit/[UUID]
    MISP Guard->>Internal MISP: [POST]/events/edit/[UUID]
    end
```

## PULL
```mermaid
sequenceDiagram
    participant External MISP
    participant MISP Guard
    participant Internal MISP

    External MISP->>MISP Guard: [GET]/servers/getVersion
    MISP Guard->>Internal MISP: [GET]/servers/getVersion
    Internal MISP->>MISP Guard: [GET]/servers/getVersion
    MISP Guard->>External MISP: [GET]/servers/getVersion

    External MISP->>+MISP Guard: [POST]/events/index
    note right of MISP Guard: Only `minimal` search requests to /events/index are allowed
    MISP Guard->>-Internal MISP: [POST]/events/index
    Internal MISP->>MISP Guard: [POST]/events/index
    MISP Guard->>External MISP: [POST]/events/index

    External MISP->>MISP Guard: [GET]/events/view/[UUID]
    MISP Guard->>Internal MISP: [GET]/events/view/[UUID]
    Internal MISP->>+MISP Guard: [GET]/events/view/[UUID]
    note right of MISP Guard: Outgoing Event is inspected and rejected with 403 if any block rule matches
    MISP Guard->>-External MISP: [GET]/events/view/[UUID]

    External MISP->>MISP Guard: [GET]/shadow_attributes/index
    MISP Guard->>Internal MISP: [GET]/shadow_attributes/index
    Internal MISP->>+MISP Guard: [GET]/shadow_attributes/index
    note right of MISP Guard: Outgoing Shadow Attributes are inspected and rejected with 403 if any block rule matches
    MISP Guard->>-External MISP: [GET]/shadow_attributes/index
```




> **NOTE: The `External MISP` server needs to have the `misp-guard` hostname configured as the server hostname you are going to pull from, **not** the `Internal MISP` hostname.**

**Supported block rules:**
* `blocked_tags`: Blocks if the event/attributes contains certain tags.
* `blocked_distribution_levels`: Blocks if the event/objects/attributes matches one of the blocked distribution levels.
  * `"0"`: Organisation Only
  * `"1"`: Community Only
  * `"2"`: Connected Communities
  * `"3"`: All Communities
  * `"4"`: Sharing Group
  * `"5"`: Inherit Event
* `blocked_sharing_groups_uuids`: Blocks if the event/objects/attributes matches one of the blocked sharing groups uuids.
* `blocked_attribute_types`: Blocks if the event contains an attribute matching one of this types.
* `blocked_attribute_categories`: Blocks if the event contains an attribute matching one of this categories.
* `blocked_object_types`: Blocks if the event contains an object matching one of this types.

See sample config [here](src/test/test_config.json).

## Instructions

### Installation
```bash
$ git clone https://github.com/MISP/misp-guard.git
$ cd src/
$ pip install -r requirements.txt
```

### Setup

1. Define your block rules in the `config.json` file.
2. Start mitmproxy with the `mispguard` addon:
    ```
    $ mitmdump -s mispguard.py -p 8888 --certs *=cert.pem --set config=config.json 
    Loading script mispguard.py
    MispGuard initialized
    running block rules: no-tlp-red-events
    Proxy server listening at *:8888
    ``` 
    _Add `-k` to accept self-signed certificates._

3. Configure the proxy in your MISP instance, set the following MISP  `Proxy.host` and `Proxy.port` settings accordingly.

Done, outgoing MISP sync requests will be inspected and dropped according to the specified block rules.


> NOTE: add `-v` to `mitmdump` to increase verbosity and display debug logs.
