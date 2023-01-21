# Argon2

This is the underlying C implementation of Argon2 that powers pyargon2.

## Usage

`make` builds the executable `argon2`, the static library `libargon2.a`,
and the shared library `libargon2.so` (or on macOS, the dynamic library
`libargon2.dylib` -- make sure to specify the installation prefix when
you compile: `make PREFIX=/usr`). Make sure to run `make test` to verify
that your build produces valid results. `sudo make install PREFIX=/usr`
installs it to your system.

### Command-line utility

`argon2` is a command-line utility to test specific Argon2 instances
on your system. To show usage instructions, run
`./argon2 -h` as
```
Usage:  ./argon2 [-h] salt [-i|-d|-id] [-t iterations] [-m memory] [-p parallelism] [-l hash length] [-e|-r] [-v (10|13)]
        Password is read from stdin
Parameters:
        salt            The salt to use, at least 8 characters
        -i              Use Argon2i (this is the default)
        -d              Use Argon2d instead of Argon2i
        -id             Use Argon2id instead of Argon2i
        -t N            Sets the number of iterations to N (default = 3)
        -m N            Sets the memory usage of 2^N KiB (default 12)
        -p N            Sets parallelism to N threads (default 1)
        -l N            Sets hash output length to N bytes (default 32)
        -e              Output only encoded hash
        -r              Output only the raw bytes of the hash
        -v (10|13)      Argon2 version (defaults to the most recent version, currently 13)
        -h              Print argon2 usage
```
For example, to hash "password" using "somesalt" as a salt and doing 2
iterations, consuming 64 MiB, using four parallel threads and an output hash
of 24 bytes
```
$ echo -n "password" | ./argon2 somesalt -t 2 -m 16 -p 4 -l 24
Type:           Argon2i
Iterations:     2
Memory:         65536 KiB
Parallelism:    4
Hash:           45d7ac72e76f242b20b77b9bf9bf9d5915894e669a24e6c6
Encoded:        $argon2i$v=19$m=65536,t=2,p=4$c29tZXNhbHQ$RdescudvJCsgt3ub+b+dWRWJTmaaJObG
0.188 seconds
Verification ok
```

### Library

`libargon2` provides an API to both low-level and high-level functions
for using Argon2.

The example program below hashes the string "password" with Argon2i
using the high-level API and then using the low-level API. While the
high-level API takes the three cost parameters (time, memory, and
parallelism), the password input buffer, the salt input buffer, and the
output buffers, the low-level API takes in these and additional parameters
, as defined in [`include/argon2.h`](include/argon2.h).

There are many additional parameters, but we will highlight three of them here.

1. The `secret` parameter, which is used for [keyed hashing](
   https://en.wikipedia.org/wiki/Hash-based_message_authentication_code).
   This allows a secret key to be input at hashing time (from some external
   location) and be folded into the value of the hash. This means that even if
   your salts and hashes are compromised, an attacker cannot brute-force to find
   the password without the key.

2. The `ad` parameter, which is used to fold any additional data into the hash
   value. Functionally, this behaves almost exactly like the `secret` or `salt`
   parameters; the `ad` parameter is folding into the value of the hash.
   However, this parameter is used for different data. The `salt` should be a
   random string stored alongside your password. The `secret` should be a random
   key only usable at hashing time. The `ad` is for any other data.

3. The `flags` parameter, which determines which memory should be securely
   erased. This is useful if you want to securely delete the `pwd` or `secret`
   fields right after they are used. To do this set `flags` to either
   `ARGON2_FLAG_CLEAR_PASSWORD` or `ARGON2_FLAG_CLEAR_SECRET`. To change how
   internal memory is cleared, change the global flag
   `FLAG_clear_internal_memory` (defaults to clearing internal memory).

Here the time cost `t_cost` is set to 2 iterations, the
memory cost `m_cost` is set to 2<sup>16</sup> kibibytes (64 mebibytes),
and parallelism is set to 1 (single-thread).

Compile for example as `gcc test.c libargon2.a -Isrc -o test`, if the program
below is named `test.c` and placed in the project's root directory.

```c
#include "argon2.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define HASHLEN 32
#define SALTLEN 16
#define PWD "password"

int main(void)
{
    uint8_t hash1[HASHLEN];
    uint8_t hash2[HASHLEN];

    uint8_t salt[SALTLEN];
    memset( salt, 0x00, SALTLEN );

    uint8_t *pwd = (uint8_t *)strdup(PWD);
    uint32_t pwdlen = strlen((char *)pwd);

    uint32_t t_cost = 2;            // 2-pass computation
    uint32_t m_cost = (1<<16);      // 64 mebibytes memory usage
    uint32_t parallelism = 1;       // number of threads and lanes

    // high-level API
    argon2i_hash_raw(t_cost, m_cost, parallelism, pwd, pwdlen, salt, SALTLEN, hash1, HASHLEN);

    // low-level API
    argon2_context context = {
        hash2,  /* output array, at least HASHLEN in size */
        HASHLEN, /* digest length */
        pwd, /* password array */
        pwdlen, /* password length */
        salt,  /* salt array */
        SALTLEN, /* salt length */
        NULL, 0, /* optional secret data */
        NULL, 0, /* optional associated data */
        t_cost, m_cost, parallelism, parallelism,
        ARGON2_VERSION_13, /* algorithm version */
        NULL, NULL, /* custom memory allocation / deallocation functions */
        /* by default only internal memory is cleared (pwd is not wiped) */
        ARGON2_DEFAULT_FLAGS
    };

    int rc = argon2i_ctx( &context );
    if(ARGON2_OK != rc) {
        printf("Error: %s\n", argon2_error_message(rc));
        exit(1);
    }
    free(pwd);

    for( int i=0; i<HASHLEN; ++i ) printf( "%02x", hash1[i] ); printf( "\n" );
    if (memcmp(hash1, hash2, HASHLEN)) {
        for( int i=0; i<HASHLEN; ++i ) {
            printf( "%02x", hash2[i] );
        }
        printf("\nfail\n");
    }
    else printf("ok\n");
    return 0;
}
```

To use Argon2d instead of Argon2i call `argon2d_hash_raw` instead of
`argon2i_hash_raw` using the high-level API, and `argon2d` instead of
`argon2i` using the low-level API. Similarly for Argon2id, call `argon2id_hash_raw`
and `argon2id`.

To produce the crypt-like encoding rather than the raw hash, call
`argon2i_hash_encoded` for Argon2i, `argon2d_hash_encoded` for Argon2d, and
`argon2id_hash_encoded` for Argon2id

See [`include/argon2.h`](include/argon2.h) for API details.

*Note: in this example the salt is set to the all-`0x00` string for the sake of simplicity, but in your application you should use a random salt*
