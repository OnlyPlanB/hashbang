# #! - Core Infrastructure #

<http://github.com/hashbang/hashbang>

## About ##

This repository contains the design documents and documentation for
[Hashbang's](https://hashbang.sh) overall infrastructure.

Likewise, its associated [issue tracker](https://github.com/hashbang/hashbang/issues)
is used for keeping track of infra-wide issues, bugs, improvements, ...

## Services ##

Currently we provide the following services:

  * SSH - ssh://hashbang.sh:22
    - [Source Code](https://github.com/hashbang/shell-server)
    - [Docker Image](https://hub.docker.com/r/hashbang/shell-server/)
  * IRC - ircs://irc.hashbang.sh:6697 
    - Server
      - [Source Code](https://github.com/hashbang/hashbang)
      - [Docker Image](https://hub.docker.com/r/hashbang/unrealircd/)
    - Services
      - [Source Code](https://github.com/hashbang/docker-anope)
      - [Docker Image](https://hub.docker.com/r/hashbang/anope/)
  * Bitlbee - ircs://im.hashbang.sh:6697 
    - [Source Code](https://github.com/hashbang/hashbang)
    - [Docker Image](https://hub.docker.com/r/hashbang/unrealircd/)
  * SMTP - smtp://mail.hashbang.sh:25
    - [Source Code](https://github.com/hashbang/docker-postfix)
    - [Docker Image](https://hub.docker.com/r/hashbang/postfix/)
  * VOIP - mumble://voip.hashbang.sh:64738
    - [Source Code](https://github.com/hashbang/docker-mumble)
    - [Docker Image](https://hub.docker.com/r/hashbang/mumble/)
  * LDAP - ldap://ldap.hashbang.sh:636
    - [Source Code](https://github.com/hashbang/docker-slapd)
    - [Docker Image](https://hub.docker.com/r/hashbang/slapd/)

## Documentation ##

  - [Abuse Prevention](https://github.com/hashbang/hashbang/tree/master/abuse)
  - [Next-Gen UserDB](https://github.com/hashbang/userdb)

## Development ##

First step to build/develop/experiment would be to spin up a local copy of our
infra and fire up a shell in it.

You can do that with:

```
make develop
```

From here you should be set to experiment with and contribute changes to any
part of the #! infrastructure.

See each individual project repo for specific development details.

## Testing ##

Though incomplete, you can run our full end-to-end test suite locally with:

```
make test
```

Please ensure all tests pass before submitting Pull Requests for any repo.

## Notes ##

  Use at your own risk. You may be eaten by a grue.

  Questions/Comments?

  Talk to us via:

  [Email](mailto://team@hashbang.sh) |
  [IRC](ircs://irc.hashbang.sh:6697/#!) |
  [Github](http://github.com/hashbang/)
