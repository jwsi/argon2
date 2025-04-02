<p align="center">
    <picture align="center">
        <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/jwsi/argon2/master/docs/static/logo.png">
        <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/jwsi/argon2/master/docs/static/logo.png">
        <img alt="argon2" src="https://raw.githubusercontent.com/jwsi/argon2/master/docs/static/logo.png" height="250px">
    </picture>
</p>

<p align="center">
    <strong>Argon2 C Reference Library</strong><br>
    A maintained fork from the original <a href="https://github.com/P-H-C/phc-winner-argon2">PHC Repo</a>
</p>

## Usage

`make` builds the executable `argon2` along with static libraries, shared
or dynamic libraries. 

`make test` verifies that your build produces valid results. 

`sudo make install` installs it to your system.

### Command-line utility

`argon2` is a command-line utility to test specific Argon2 instances
on your system. To show usage instructions, run
`./argon2 -h`
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

For example, to hash "a strong password" using "a unique salt" using the following parameters:

- Variant = `id`
- Hash Length (bytes) = 32
- Iterations = 620
- Memory usage = 4096 bytes (2^12)
- Threads = 8

Execute:
```
$ echo -n "a strong password" | ./argon2 "a unique salt" -id -m 12 -p 8 -t 620 -l 32
Type:       Argon2id
Iterations: 620
Memory:     4096 KiB
Parallelism:    8
Hash:       5fffc57a27340c7c313def2c9be00bec9a9db36aa5787ef49876a1bf195c7bdf
Encoded:    $argon2id$v=19$m=4096,t=620,p=8$YSB1bmlxdWUgc2FsdA$X//Feic0DHwxPe8sm+AL7Jqds2qleH70mHahvxlce98
1.561 seconds
Verification ok
```
