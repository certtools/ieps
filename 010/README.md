# IEP010: Shadowserver schema update (2025-01-14)

The following schema changes are open for discussion and scheduled to be committed on 2025-01-28.

## Change summary

 * Added scan_ip_tunnel and scan6_ip_tunnel reports.
 * Revised the feed_name and url for the scan_msrpc report.
 * Added scan6_rsync report.

## Details

### scan_ip_tunnel:

```
{
   "constant_fields" : {
      "classification.identifier" : "open-ip-tunnel",
      "classification.taxonomy" : "vulnerable",
      "classification.type" : "vulnerable-system"
   },
   "feed_name" : "Open-IP-Tunnel",
   "file_name" : "scan_ip_tunnel",
   "optional_fields" : [
      [
         "extra.",
         "severity",
         "validate_to_none"
      ],
      [
         "protocol.transport",
         "protocol"
      ],
      [
         "source.reverse_dns",
         "hostname"
      ],
      [
         "extra.",
         "tag",
         "validate_to_none"
      ],
      [
         "source.asn",
         "asn",
         "invalidate_zero"
      ],
      [
         "source.geolocation.cc",
         "geo"
      ],
      [
         "source.geolocation.region",
         "region"
      ],
      [
         "source.geolocation.city",
         "city"
      ],
      [
         "extra.source.naics",
         "naics",
         "invalidate_zero"
      ],
      [
         "extra.",
         "hostname_source",
         "validate_to_none"
      ],
      [
         "extra.source.sector",
         "sector",
         "validate_to_none"
      ],
      [
         "extra.",
         "response",
         "validate_to_none"
      ]
   ],
   "required_fields" : [
      [
         "time.source",
         "timestamp",
         "add_UTC_to_timestamp"
      ],
      [
         "source.ip",
         "ip",
         "validate_ip"
      ],
      [
         "source.port",
         "port",
         "convert_int"
      ]
   ],
   "url" : "https://www.shadowserver.org/what-we-do/network-reporting/open-ip-tunnel-report"
}
```

### scan6_ip_tunnel:

```
{
   "constant_fields" : {
      "classification.identifier" : "open-ip-tunnel",
      "classification.taxonomy" : "vulnerable",
      "classification.type" : "vulnerable-system"
   },
   "feed_name" : "IPv6-Open-IP-Tunnel",
   "file_name" : "scan6_ip_tunnel",
   "optional_fields" : [
      [
         "extra.",
         "severity",
         "validate_to_none"
      ],
      [
         "protocol.transport",
         "protocol"
      ],
      [
         "source.reverse_dns",
         "hostname"
      ],
      [
         "extra.",
         "tag"
      ],
      [
         "source.asn",
         "asn",
         "invalidate_zero"
      ],
      [
         "source.geolocation.cc",
         "geo"
      ],
      [
         "source.geolocation.region",
         "region"
      ],
      [
         "source.geolocation.city",
         "city"
      ],
      [
         "extra.source.naics",
         "naics",
         "invalidate_zero"
      ],
      [
         "extra.",
         "hostname_source",
         "validate_to_none"
      ],
      [
         "extra.source.sector",
         "sector",
         "validate_to_none"
      ],
      [
         "extra.",
         "response",
         "validate_to_none"
      ]
   ],
   "required_fields" : [
      [
         "time.source",
         "timestamp",
         "add_UTC_to_timestamp"
      ],
      [
         "source.ip",
         "ip",
         "validate_ip"
      ],
      [
         "source.port",
         "port",
         "convert_int"
      ]
   ],
   "url" : "https://www.shadowserver.org/what-we-do/network-reporting/open-ip-tunnel-report"
}
```

### scan_msrpc:

```
*** /tmp/imq.cur	2025-01-14 20:58:19.994103503 +0000
--- /tmp/imq.new	2025-01-14 20:58:19.998103811 +0000
***************
*** 4,10 ****
        "classification.taxonomy" : "vulnerable",
        "classification.type" : "potentially-unwanted-accessible"
     },
!    "feed_name" : "Accessible-MS-RPC-Endpoint-Mapper",
     "file_name" : "scan_msrpc",
     "optional_fields" : [
        [
--- 4,10 ----
        "classification.taxonomy" : "vulnerable",
        "classification.type" : "potentially-unwanted-accessible"
     },
!    "feed_name" : "Accessible-MS-RPC",
     "file_name" : "scan_msrpc",
     "optional_fields" : [
        [
***************
*** 135,140 ****
           "convert_int"
        ]
     ],
!    "url" : "https://www.shadowserver.org/what-we-do/network-reporting/ms-rpc-endpoint-mapper-report"
  }
  
--- 135,140 ----
           "convert_int"
        ]
     ],
!    "url" : "https://www.shadowserver.org/what-we-do/network-reporting/accessible-ms-rpc-service-report/"
  }
```

### scan6_rsync

```
{
   "constant_fields" : {
      "classification.identifier" : "accessible-rsync",
      "classification.taxonomy" : "vulnerable",
      "classification.type" : "vulnerable-system",
      "protocol.application" : "rsync"
   },
   "feed_name" : "IPv6-Accessible-Rsync",
   "file_name" : "scan6_rsync",
   "optional_fields" : [
      [
         "extra.",
         "severity",
         "validate_to_none"
      ],
      [
         "protocol.transport",
         "protocol"
      ],
      [
         "source.reverse_dns",
         "hostname"
      ],
      [
         "extra.",
         "tag"
      ],
      [
         "source.asn",
         "asn",
         "invalidate_zero"
      ],
      [
         "source.geolocation.cc",
         "geo"
      ],
      [
         "source.geolocation.region",
         "region"
      ],
      [
         "source.geolocation.city",
         "city"
      ],
      [
         "extra.source.naics",
         "naics",
         "invalidate_zero"
      ],
      [
         "extra.",
         "hostname_source",
         "validate_to_none"
      ],
      [
         "extra.source.sector",
         "sector",
         "validate_to_none"
      ],
      [
         "extra.",
         "module",
         "validate_to_none"
      ],
      [
         "extra.",
         "motd",
         "validate_to_none"
      ],
      [
         "extra.",
         "has_password",
         "validate_to_none"
      ]
   ],
   "required_fields" : [
      [
         "time.source",
         "timestamp",
         "add_UTC_to_timestamp"
      ],
      [
         "source.ip",
         "ip",
         "validate_ip"
      ],
      [
         "source.port",
         "port",
         "convert_int"
      ]
   ],
   "url" : "https://www.shadowserver.org/what-we-do/network-reporting/accessible-rsync-report/"
}
```
