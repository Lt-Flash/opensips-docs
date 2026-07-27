---
title: "Using the Compression module"
subtitle: "Compression"
subtitleHref: "/modules/devel/compression"
description: "This tutorial explains how to use the compression module to compact, compress and decompress SIP messages."
---

## Tutorial Overview

The purpose of this tutorial is to help you understand how the **compression** module works and how it should be used. After reading
this tutorial you should understand which functions this module provides and how to use them in order to reduce the size of your SIP messages.

## Current features

As stated in the [documentation](/modules/devel/compression), the module currently features:
* **Message compaction**
  * headers not specified in the given whitelist will be removed, except for mandatory headers such as Via, To and From
  * some headers which have a compact form (not all of them) will be reduced to short form (eg. Via --> `v`; Content-Length --> `l`)
  * multiple headers of same type will be reduced to one header, containing all of them (eg. Header1: text1 and Header1: text2 reduced to Header1: text1, text2)
  * unnecessary SDP body `a=rtpmap:CODEC_NO` lines where `CODEC_NO < 98` will be removed
* **Message compression/decompression**
  * compression of the message body
  * compression of headers not specified in the whitelist, except for mandatory headers
  * base64 encoding

## Example script
The following script implements this scenario: two directly connected SIP servers. One of them does compaction and compression, then forwards the message. The other one receives the compressed message, decompresses it and then forwards it.

SIP logic provided for compressing server:
* message compression
* message compaction

SIP logic provided for decompressing server:
* message decompression

**The compressing server**

Must load the module

```opensips
loadmodule "compression.so"
```

Optionally, we can set the compression level (1-9), default 6:
```opensips
modparam("compression", "compression_level", 7)
```

And then let's assume that for all INVITEs we want to do compression and compaction:

```opensips
route {
      if (is_method("INVITE")) {
         /*
             we want body and headers compression separately
             will use deflate
             won't use base64
             let's say the headers we want to compress are Header1 and Header2
         */
         if (!mc_compress(0, "bhs", "Header1|Header2")) {
             xlog("compression failed\n");
             exit;
         }
         /*
            now do compaction but keep User-Agent and P-Asserted-Identity headers
            the rest of them will be removed (excepting the mandatory ones)
            VERY IMPORTANT: keep in mind that if you do compaction after compression
            3 new headers might be added: Content-Encoding (if you compress the body),
            Comp-Hdrs(the compressed headers), Headers-Encoding(the algorithm used to
            compress the headers in Comp-Hdrs). These headers will not be kept by
            OpenSIPS, so you must explicitly specify them in mc_compact whitelist
            of headers not to be removed.
         */
         if (!mc_compact("User-Agent|P-Asserted-Identity|Content-Encoding|Comp-Hdrs|Headers-Encoding")) {
             xlog("Compaction failed\n");
             exit;
         }
         $var(dest) = "SO.ME.IP.ADDRESS:SOME_PORT";
         forward("$var(dest)");
      }
}
```

**Now let's go to the receiving server, the one doing the decompression.**

Assume the receiving server has also loaded "compression.so". The compressing server does compression for REGISTERs, so the decompressing server will do the same. Keep in mind that decompression should be one of the first things you do for a compressed message, because this function replaces the SIP message buffer and any earlier changes to the message will not be taken into consideration.

```opensips
route {
      if (is_method("REGISTER")) {
          if (!mc_decompress()) {
              xlog("decompression failed\n");
              exit;
          }
          /* use the decompressed message */
          #some_instructions#
          /* forward it */
          $var(endpoint_addr) = "SO.ME.IP.ADDRESS";
          forward("$var(endpoint_addr)");
      }
}
```
