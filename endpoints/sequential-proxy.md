---
lastmod: 2019-09-26
date: 2018-11-11
toc: true
linktitle: Sequential Proxy (chain reqs.)
title: Sequential Proxy
since: 0.7
notoc: true
weight: 60
images:
- /images/documentation/krakend-sequential-call.png
menu:
  community_current:
    parent: "040 Endpoint Configuration"
meta:
  noop_incompatible: true
  since: 0.7
  source: https://github.com/luraproject/lura
  namespace:
  - proxy
  scope:
  - endpoint
  - async_agent
  log_prefix:
  - "[SERVICE: Gin]"
---
The best experience consumers can have with KrakenD API is by letting the system fetch all the data from the different backends concurrently at the same time. However, there are times when you need to **delay a backend call** until you can inject as input the result of a previous call.

The sequential proxy allows you to **chain backend requests**.

## Chaining the requests
All you need to enable the sequential proxy is add in the endpoint definition the following configuration:

```json
{
    "endpoint": "/hotels/{id}",
    "extra_config": {
          "proxy": {
              "sequential": true
          }
      }
}
```


When the sequential proxy is enabled, the `url_pattern` of every backend can use a new variable that references the **resp**onse of a previous API call. The variable has the following construction:

```js
{resp0_XXXX}
```



Where `0` is the index of the specific backend you want to access ( `backend` array), and where `XXXX` is the attribute name you want to inject from the previous call.

{{< note title="Unsafe operations" >}}
If you use unsafe methods (not a `GET`), they can only be placed in the last position of the sequence. Sequences are meant to be used in read-only operations except for the last call. Sequence is not meant to be used in distributed transactions.
{{< /note >}}

## Example
It's easier to understand with the example of the graph:

KrakenD calls a backend `/hotels/{hotel_id}` that returns data for the requested hotel. When we request for the hotel ID `25` the backend service responds with the hotel data, including a `destination_id` that is a relationship identifier. The output for `GET /hotels/25` is like the following:

```json
{
    "hotel_id": 25,
    "name": "Hotel California",
    "destination_id": 1034
}
```


KrakenD waits for the response of the backend and looks for the field `destination_id`. And then injects the value in the next backend call to `/destinations/{destination_id}`. In this case the next call is `GET /destinations/1034`, and the response is:

```json
{
    "destination_id": 1034,
    "destinations": [
        "LAX",
        "SFO",
        "OAK"
    ]
}
```


Now KrakenD has both responses from the backends and can merge the data, returning the following object to the user:

```json
{
    "hotel_id": 25,
    "name": "Hotel California",
    "destination_id": 1034,
    "destinations": [
        "LAX",
        "SFO",
        "OAK"
    ]
}
```


The configuration needed for this example is:

```json
{
    "endpoint": "/hotel-destinations/{id}",
    "backend": [
        { <--- Index 0
            "host": [
                "https://hotels.api"
            ],
            "url_pattern": "/hotels/{id}"
        },
        { <--- Index 1
            "host": [
                "https://destinations.api"
            ],
            "url_pattern": "/destinations/{resp0_destination_id}"
        }
    ],
    "extra_config": {
        "proxy": {
            "sequential": true
        }
    }
}
```



The key here is the variable `{resp0_destination_id}` that refers to `destination_id` for the backend with index `0` (first in the list).
