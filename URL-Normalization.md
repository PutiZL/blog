<!--
author: zhuoliang
head: http://pingodata.qiniudn.com/jockchou-avatar.jpg
date: 2016-08-01
title: URL Normalization
tags: Web
category: Web
status: publish
-->

## Normalization process ##
**URL normalization** is the process by which URLs are modified and standardized in a consistent manner. The goal of the normalization process is to transform a URL into a normalized URL so it is possible to determine if two syntactically different URLs may be equivalent.

Search engines employ URL normalization in order to assign importance to web pages and to reduce indexing of duplicate pages. Web crawlers perform URL normalization in order to avoid crawling the same resource more than once. Web browsers may perform normalization to determine if a link has been visited or to determine if a page has been cached.

There are several types of normalization that may be performed. Some of them are always semantics preserving and some may not be.

###  Normalizations that preserve semantics  ###

The following normalizations are described in [RFC 3986](https://tools.ietf.org/html/rfc3986) to result in equivalent URLs:

**Converting the scheme and host to lower case.** The scheme and host components of the URL are case-insensitive. Most normalizers will convert them to lowercase. Example:

    
    HTTP://www.Example.com/ → http://www.example.com/ 

**Capitalizing letters in escape sequences.** All letters within a percent-encoding triplet (e.g., "%3A") are case-insensitive, and should be capitalized. Example:

	
    http://www.example.com/a%c2%b1b → http://www.example.com/a%C2%B1b 

**Decoding percent-encoded octets of unreserved characters.** For consistency, percent-encoded octets in the ranges of ALPHA (`%41–%5A` and `%61–%7A`), DIGIT (`%30–%39`), hyphen (`%2D`), period (`%2E`), underscore (`%5F`), or tilde (`%7E`) should not be created by URI producers and, when found in a URI, should be decoded to their corresponding unreserved characters by URI normalizers. Example:

	
    http://www.example.com/%7Eusername/ → http://www.example.com/~username/ 

Removing the default port. The default port (port 80 for the `http` scheme) may be removed from (or added to) a URL. Example:

    
    http://www.example.com:80/bar.html → http://www.example.com/bar.html 

### Normalizations that usually preserve semantics ###

For http and https URLs, the following normalizations listed in RFC 3986 may result in equivalent URLs, but are not guaranteed to by the standards:

**Adding trailing / Directories** are indicated with a trailing slash and should be included in URLs. Example:

    http://www.example.com/alice → http://www.example.com/alice/

However, there is no way to know if a URL path component represents a directory or not. [RFC 3986](https://tools.ietf.org/html/rfc3986) notes that if the former URL redirects to the latter URL, then that is an indication that they are equivalent.

**Removing dot-segments.** The segments `..` and `.` can be removed from a URL according to the algorithm described in [RFC 3986](https://tools.ietf.org/html/rfc3986) (or a similar algorithm). Example:

	http://www.example.com/../a/b/../c/./d.html → http://www.example.com/a/c/d.html

However, if a removed `..` component, e.g. `b/..`, is a symlink to a directory with a different parent, eliding `b/..` will result in a different path and URL. In rare cases depending on the web server, this may even be true for the root directory (e.g. `//www.example.com/..` may not be equivalent to `//www.example.com/`.

### Normalizations that change semantics ###
Applying the following normalizations result in a semantically different URL although it may refer to the same resource:


**Removing directory index.** Default directory indexes are generally not needed in URLs. Examples:
	

    http://www.example.com/default.asp → http://www.example.com/
    http://www.example.com/a/index.html → http://www.example.com/a/ 

**Removing the fragment** The fragment component of a URL is never seen by the server and can sometimes be removed. Example:

    http://www.example.com/bar.html#section1 → http://www.example.com/bar.html

However, AJAX applications frequently use the value in the fragment.


**Replacing IP with domain name.** Check if the IP address maps to a domain name. Example:

	http://208.77.188.166/ → http://www.example.com/

The reverse replacement is rarely safe due to virtual web servers.



**Limiting protocols.** Limiting different application layer protocols. For example, the “https” scheme could be replaced with “http”. Example:

    https://www.example.com/ → http://www.example.com/ 

**Removing duplicate slashes** Paths which include two adjacent slashes could be converted to one. Example:
	
    http://www.example.com/foo//bar.html → http://www.example.com/foo/bar.html 

**Removing or adding *www* as the first domain label.** Some websites operate identically in two Internet domains: one whose least significant label is “www” and another whose name is the result of omitting the least significant label from the name of the first, the latter being known as a naked domain. For example, `http://example.com/` and `http://www.example.com/` may access the same website. Many websites redirect the user from the www to the non-www address or vice versa. A normalizer may determine if one of these URLs redirects to the other and normalize all URLs appropriately. Example:
	
    http://www.example.com/ → http://example.com/ 

**Sorting the query parameters.** Some web pages use more than one query parameter in the URL. A normalizer can sort the parameters into alphabetical order (with their values), and reassemble the URL. Example:

	http://www.example.com/display?lang=en&article=fred → http://www.example.com/display?article=fred&lang=en

However, the order of parameters in a URL may be significant (this is not defined by the standard) and a web server may allow the same variable to appear multiple times.

**Removing unused query variables.** A page may only expect certain parameters to appear in the query; unused parameters can be removed. Example:


    http://www.example.com/display?id=&sort=ascending → http://www.example.com/display 

**Removing the *?* when the query is empty.** When the query is empty, there may be no need for the "?". Example:

    http://www.example.com/display? → http://www.example.com/display 


 
## Open Solutions ##

### Snort ###

    typedef struct s_HI_CLIENT_REQ
	{
   		/*
    	u_char *method;
    	int  method_size;
    	*/

		const u_char *uri;
   		const u_char *uri_norm;
    	const u_char *post_raw;
    	const u_char *post_norm;
    	const u_char *header_raw;
    	const u_char *header_norm;
    	COOKIE_PTR cookie;
    	const u_char *cookie_norm;
   		const u_char *method_raw;

   		u_int uri_size;
    	u_int uri_norm_size;
    	u_int post_raw_size;
    	u_int post_norm_size;
    	u_int header_raw_size;
    	u_int header_norm_size;
    	u_int cookie_norm_size;
    	u_int method_size;

    	/*
    	u_char *param;
    	u_int  param_size;
   		u_int  param_norm;
    	*/

    	/*
    	u_char *ver;
    	u_int  ver_size;

    	u_char *hdr;
    	u_int  hdr_size;

    	u_char *payload;
    	u_int  payload_size;
    	*/

    	const u_char *pipeline_req;
    	u_char method;
    	uint16_t uri_encode_type;
    	uint16_t header_encode_type;
    	uint16_t cookie_encode_type;
    	uint16_t post_encode_type;
    	const u_char *content_type;

	}  HI_CLIENT_REQ;




