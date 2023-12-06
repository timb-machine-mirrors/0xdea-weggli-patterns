# weggli-patterns
[![](https://img.shields.io/github/stars/0xdea/weggli-patterns.svg?color=yellow)](https://github.com/0xdea/weggli-patterns)
[![](https://img.shields.io/github/forks/0xdea/weggli-patterns.svg?color=green)](https://github.com/0xdea/weggli-patterns)
[![](https://img.shields.io/github/watchers/0xdea/weggli-patterns.svg?color=red)](https://github.com/0xdea/weggli-patterns)
[![](https://img.shields.io/badge/twitter-%400xdea-blue.svg)](https://twitter.com/0xdea)
[![](https://img.shields.io/badge/mastodon-%40raptor-purple.svg)](https://infosec.exchange/@raptor)

> "No one gives a s*** about the old scene people anymore I’m sure,  
> bunch of old a** people grepping for the last of the memcpy." 
> 
> -- Bas Alberts

A collection of my weggli patterns to facilitate vulnerability research.

See also:  
https://github.com/weggli-rs/weggli  
https://dustri.org/b/playing-with-weggli.html  
https://github.com/plowsec/weggli-patterns  
https://github.com/synacktiv/Weggli_rules_SSTIC2023  
https://twitter.com/richinseattle/status/1729654184633327720  

## buffer overflows

### call to insecure API functions (CWE-120, CWE-242, CWE-676)
```
weggli -R 'func=^gets' '{$func(_);}' .
weggli -R 'func=^st(r|p)(cpy|cat)' '{$func(_);}' .
weggli -R 'func=^wc(s|p)(cpy|cat)' '{$func(_);}' .
weggli -R 'func=sprintf$' '{$func(_);}' .
weggli -R 'func=scanf$' '{$func(_);}' .
```

### incorrect use of strncat (CWE-193, CWE-787)
```
weggli '{strncat(_,_,sizeof(_));}' .
weggli '{strncat(_,_,strlen(_));}' .
weggli '{strncat($dst,$src,sizeof($dst)-strlen($dst));}' .

# this won't work due to current limitations in the query language
# weggli '{_ $buf[$len]; strncat($buf,_,$len);}' .
# https://github.com/weggli-rs/weggli/issues/59
```

### destination buffer access using size of source buffer (CWE-806)
```
weggli -R 'func=cpy' '{$func(_,$src,_($src));}' .

# this won't work due to current limitations in the query language
# weggli -R 'func=cpy' '{_ $src[$len]; $func($dst,$src,$len);}' .
# https://github.com/weggli-rs/weggli/issues/59
```

### use of sizeof() on a pointer type (CWE-467)
```
weggli '{_* $p; sizeof($p);}' .
weggli '{_* $p=_; sizeof($p);}' .
weggli '_ $func(_* $p) {sizeof($p);}' .
```

### lack of explicit NUL-termination after strncpy(), etc. (CWE-170)
```
weggli -R 'func=ncpy' '{$func($buf,_); not: $buf[_]=_;}' .

# some variants: memcpy, read, readlink, fread, etc.
```

### off-by-one error (CWE-193)
```
weggli '{$buf[sizeof($buf)];}' .
weggli '{_ $buf[$len]; $buf[$len]=_;}' .
weggli '{strlen($src)>sizeof($dst);}' .
weggli '{strlen($src)<=sizeof($dst);}' .
weggli '{sizeof($dst)<strlen($src);}' .
weggli '{sizeof($dst)>=strlen($src);}' .
weggli '{$buf[strlen($buf)-1];}' .
weggli '{malloc(strlen($buf));}' .
```

### use of pointer subtraction to determine size (CWE-469)
```
weggli '{_* $p1; $p1-$p2;}' .
weggli '{_* $p2; $p1-$p2;}' .
weggli '{_* $p1=_; $p1-$p2;}' .
weggli '{_* $p2=_; $p1-$p2;}' .
weggli '_ $func(_* $p1) {$p1-$p2;}' .
weggli '_ $func(_* $p2) {$p1-$p2;}' .
```

### potentially unsafe use of the return value of snprintf(), etc. (CWE-787)
```
weggli -R 'func=(nprintf|lcpy|lcat)' '{$ret=$func(_);}' .
```

### direct write into buffer allocated on the stack (CWE-121)
```
weggli -R 'func=(cpy|cat|memmove|memset|sn?printf)' '{_ $buf[_]; $func($buf,_);}' .
weggli '{_ $buf[_]; $buf[_]=_;}' .

# some variants: bcopy, gets, fgets, getwd, getcwd, fread, read, pread, recv, recvfrom, etc.
```

## integer overflows

### incorrect unsigned comparison (CWE-697)
```
weggli -R '$type=(unsigned|size_t)' '{$type $var; $var<0;}' .
weggli -R '$type=(unsigned|size_t)' '{$type $var; $var<=0;}' .
weggli -R '$type=(unsigned|size_t)' '{$type $var; $var>=0;}' .
weggli -R '$type=(unsigned|size_t)' '{$type $var=_; $var<0;}' .
weggli -R '$type=(unsigned|size_t)' '{$type $var=_; $var<=0;}' .
weggli -R '$type=(unsigned|size_t)' '{$type $var=_; $var>=0;}' .
```

### signed/unsigned conversion (CWE-195, CWE-196)
```
weggli -R '$copy=(cpy|ncat)' '{int $len; $copy(_,_,$len);}' .
weggli -R '$copy=(cpy|ncat)' '{int $len=_; $copy(_,_,$len);}' .
weggli -R '$copy=(cpy|ncat)' '_ $func(int $len) {$copy(_,_,$len);}' .

weggli -R '$copy=nprintf' '{int $len; $copy(_,$len);}' .
weggli -R '$copy=nprintf' '{int $len=_; $copy(_,$len);}' .
weggli -R '$copy=nprintf' '_ $func(int $len) {$copy(_,$len);}' .

weggli -R '$type=(unsigned|size_t)' '{$type $var1; int $var2; $var2=_($var1);}' .
weggli -R '$type=(unsigned|size_t)' '{$type $var1; int $var2; $var1=_($var2);}' .
weggli -R '$type=(unsigned|size_t)' '{$type $var1; int $var2=_($var1);}' .
weggli -R '$type=(unsigned|size_t)' '{int $var1; $type $var2; $var2=_($var1);}' .
weggli -R '$type=(unsigned|size_t)' '{int $var1; $type $var2; $var1=_($var2);}' .
weggli -R '$type=(unsigned|size_t)' '{int $var1=_; $type $var2=_($var1);}' .

weggli -R '$type=(unsigned|size_t)' '_ $func(int $var2) {$type $var1; $var1=_($var2);}' .
weggli -R '$type=(unsigned|size_t)' '_ $func(int $var2) {$type $var1=_($var2);}' .

weggli -R '$type=(unsigned|size_t)' '$type $func(_) {int $var; return $var;}' .
weggli -R '$type=(unsigned|size_t)' 'int $func(_) {$type $var; return $var;}' .

# there are many possible variants...
```

### integer truncation (CWE-197)
```
weggli -R 'type=(short|int|long)' '{$type $large; char $narrow; $narrow = $large; }' .
weggli -R 'type=(short|int|long)' '{$type $large; char $narrow = $large; }' .
weggli -R 'type=(int|long)' '{$type $large; short $narrow; $narrow = $large; }' .
weggli -R 'type=(int|long)' '{$type $large; short $narrow = $large; }' .
weggli '{long $large; int $narrow; $narrow = $large; }' .
weggli '{long $large; int $narrow = $large; }' .

weggli -R 'type=(short|int|long)' '_ $func($type $large) {char $narrow; $narrow = $large; }' .
weggli -R 'type=(short|int|long)' '_ $func($type $large) {char $narrow = $large; }' .
weggli -R 'type=(int|long)' '_ $func($type $large) {short $narrow; $narrow = $large; }' .
weggli -R 'type=(int|long)' '_ $func($type $large) {short $narrow = $large; }' .
weggli '_ $func(long $large) {int $narrow; $narrow = $large; }' .
weggli '_ $func(long $large) {int $narrow = $large; }' .

# there are many possible variants...
```

### signed or short sizes, lengths, offsets, counts (CWE-190, CWE-680)
```
weggli '{short _;}' .
weggli '{int _;}' .

# some variants: short int, unsigned short, unsigned short int, int
```

### casting the return value of strlen(), wcslen() to short (CWE-190, CWE-680)
```
weggli -R 'func=(str|wcs)len' '{short $len; $len=$func(_);}' .

# some variants: short int, unsigned short, unsigned short int
```

### integer wraparound (CWE-128, CWE-131, CWE-190)

TBD

## format strings

### find printf(), scanf(), syslog() family functions (CWE-134)
```
weggli -R 'func=(printf$|scanf$|syslog$)' '{$func(_);}' .

# some variants: printk, warn, vwarn, warnx, vwarnx, err, verr, errx, verrx, warnc, vwarnc, errc, verrc
```

## memory management

TBD

### use of uninitialized pointers (CWE-457, CWE-824, CWE-908)
```
weggli '{_* $p; not: $p =_; not: $func(&$p); _($p);}' .
```

## command injection

TBD

## race conditions

TBD

## privilege management

TBD

### unchecked return code of setuid() and seteuid() (CWE-252)
```
weggli -R 'func=sete?uid' '{strict: $func(_);}' .
```

## miscellaneous

TBD

### command-line argument or environment variable access
```
weggli -R 'var=(argv|envp)' '{$var[_];}' .
```
