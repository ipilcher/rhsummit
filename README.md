## To build HTML

### Setup grip

```
$ cd
$ git clone https://github.com/joeyespo/path-and-address.git
$ git clone https://github.com/joeyespo/grip.git

$ cd rhsummit

$ ln -s ~/path-and-address/path_and_address .
$ ln -s ~/grip/grip .

$ cat << 'EOF' > grip.py
#!/usr/bin/python2

# -*- coding: utf-8 -*-
import re
import sys

from grip import main

if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
    sys.exit(main())
EOF
```

### Export & post-process

```
$ ./grip.py guide.md --export
$ sed -i -e 's|@@|<b>|g' -e 's|@/|</b>|g' guide.html
```
