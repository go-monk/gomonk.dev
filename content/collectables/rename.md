+++
date = '1990-07-30'
author = 'Larry Wall'
runnable = true # a complete script; the shebang is trimmed from the excerpt
title = 'rename'
+++

Larry Wall's bulk file renamer. The renaming rule is not an option or a pattern language; it is any Perl expression that edits `$_`, so `rename 's/\.bak$//' *.bak` and `rename 'y/A-Z/a-z/' *` both just work.

```perl
($op = shift) || die "Usage: rename perlexpr [filenames]\n";
if (!@ARGV) {
    @ARGV = <STDIN>;
    chop(@ARGV);
}
for (@ARGV) {
    $was = $_;
    eval $op;
    die $@ if $@;
    rename($was,$_) unless $was eq $_;
}
```

Source: [eg/rename in the Perl distribution](https://github.com/Perl/perl5/blob/perl-5.003/eg/rename)
