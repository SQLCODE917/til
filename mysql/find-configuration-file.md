# Find The Configuration File MySQL Uses

```
mysql --help | grep Default -A 1
```

The `-A` flag prints a number of lines of training context after matching lines.
Places a line containing a group separator (--) between contiguous groups of matches.
With the `-o` or `--only-matching` option, this has no effect and a warning is given.

Expect to see something like this in the output:

```
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/local/etc/my.cnf ~/.my.cnf 
```
