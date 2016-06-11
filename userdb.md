# #! user database -- Requirements

## Data

The following data needs to be associated to a user:
- `id` (`uid == gid`)
- `name`
- `host`
- some non-relational data:
  - SSH keys.

Moreover, the DB needs to keep track of per-server information:
- `hostname` (must);
- ip, location (for display in <https://hashbang.sh/server/stats>).


## Requirements

### Immediate

These requirements **must** be achieved before deployment:

- Availability:
  - Users do not lose access to the shell servers if part of the infra goes down.
  - Loss of any part of the infrastructure is recoverable with limited data loss.
- Consistency
- Maintainability
- Privilege separation


### Long-term

These requirements **must** be *achievable*:

- Privilege separation
  - Unprivileged `hashbangctl` can only modify the user's own data
- Remote service authentication: local (shell) users should be able to authenticate
  transparently and securely to remote (#!) services (SMTP, IRC, ...).

## Services

The following services need to interact with the user DB:
- OpenSSH, through `AuthorizedKeysCommand`;
- `mail.hashbang.sh` needs to extract host info for mail routing;
- `hashbang.sh` needs to extract statistics;
- user creation.


# #! user database -- PostgreSQL-based proposal

## Design goals

This design leans strongly towards consistency of the data, enforced
as much as possible at the database level.

Part of this appears in the user of foreign keys, range or value constraints,
preventing applications from inserting (or modifying) data that violates those
constraints.

Less apparent manifestations appear in the design of the database schema:
- User's primary groups are known to have the same id and name as the users,
  and as such are not stored explicitely; such an inconsistency caused the
  [“group 3000” bug](https://github.com/hashbang/provisor/pull/25).
- Data duplication is systematically avoided, as it is a major cause of
  inconsistencies in databases; the schema is even in
  [project-join normal form](https://en.wikipedia.org/wiki/Fifth_normal_form).


## Replication

In a single-master deployment, PostgreSQL has hot replication features that
allow to immediately propagate changes to (a configurable number of) replicas,
possibly before the change is commited on the master.

Using a local PostgreSQL instance on each shell server, acting as a local replica,
immediately fulfills the availability requirements:
- each server holds a read-only copy of the database, so users can login
  regardless of whether the DB master is available;
- should the DB master be lost, the most up-to-date replica of the DB (can be found
  by comparing `pg_last_xlog_receive_location` values) can be either promoted to
  the role of master, or (preferred) copied to the new master instance.

Moreover, single-master PostgreSQL provides the usual ACID consistency guarantees.


## Database schema

The database holds two tables, `passwd` and `hosts`.

DB constraints are used to enforce, as much as possible, consistency:
- uids must be valid and unique;
- usernames must be unique and follow the proper syntax rules;
- a user's host must exist.

User records have an optional `data` column, that can hold
  additional, non-relational data as a (binary-encoded) JSON object.

```postgres
CREATE SEQUENCE user_id MINVALUE 4000 MAXVALUE 2147483647 NO CYCLE;
CREATE SEQUENCE host_id MINVALUE    0 MAXVALUE 2147483647 NO CYCLE;

CREATE DOMAIN username_t varchar(64) CHECK (
  VALUE ~ '^[a-z][a-z0-9]+$'
);

CREATE TABLE "hosts" (
  "id" integer PRIMARY KEY MINVALUE 0 DEFAULT nextval('host_id'),
  "name" varchar(10) PRIMARY KEY,
  "location" text
)

CREATE TABLE "passwd" (
  "uid" integer PRIMARY KEY MINVALUE 1000 DEFAULT nextval('user_id'),
  "name" username_t PRIMARY KEY,
  "host" integer NOT NULL REFERENCES hosts (id),
  "homedir" varchar(256) NOT NULL,
  "data" jsonb
);

CREATE TABLE "group" (
  "gid" integer PRIMARY KEY MAXVALUE 999,
  "name" username_t PRIMARY KEY,
);

CREATE TABLE "aux_groups" (
  "uid" int4 NOT NULL REFERENCES passwd (uid) ON DELETE CASCADE,
  "gid" int4 NOT NULL REFERENCES group  (gid) ON DELETE CASCADE,
  PRIMARY KEY ("uid", "gid"),
);
```

*NOTE:* Rows in `group` and `passwd` shouldn't share a `name`.
        Can this be expressed as a constraint?


Lastly, the `data` JSON object holds non-relational data.
It can be extended by users, but in any case must obey the following schema:

```yaml
$schema: http://json-schema.org/schema#
title: Manifest schema for the debconf plugin
type: object
properties:
  ssh_keys:
    type: array
    items: {type: string}
    uniqueItems: True
    description: SSH keys for the shell servers
  name:
    type: string
    description: User's name
  shell:
    type: string
    description: User's shell
  required: [ssh_keys, shell]
```

*NOTE:* It might be possible to enforce the JSON Schema in the database itself.
        This isn't an immediate goal.

*NOTE:* Yes, I'm aware I serialized the JSON Schema as YAML.  Yes, it's legit.


## Permissions

Moreso than separating permissions on a per-server basis, permissions should be
assigned on a per-service basis, and follow the least privilege principle.


### Shell servers

A shell server hosts several components that get different access rights to the DB:
- `pgsql`: the DB server itself need a DB user with the `replication` privilege.
  It gives complete read access to the database (from the master), and nothing else.
- `ssh`: needs read access to the `passwd.{name,data}` columns.
- `nss`: needs read access to `passwd`, `group` and `aux_groups`.
- `hashbangctl`: needs write access to the `passwd.data` column.


### `hashbang.sh`

The website fulfills two complementary (and independent) roles:
- user creation: `INSERT` privilege in the `passwd` table;
- statistics: read-only access to a `hosts_stats` view, created as follows:

```postgres
CREATE VIEW hosts_stats AS
  SELECT hosts.id, hosts.name, agg.count FROM hosts
  JOIN (SELECT host, count(distinct id) as count GROUP BY host) AS agg
  ON agg.host = hosts.id
```


### `mail.hashbang.sh`

The mail server only needs read access to `passwd.{name,host}` and `hosts`.


## Service integration

### Shell servers

On the shell servers, integrating the new auth DB involves three things:
- having Postgres installed and configured for streaming replication;
- having `libnss-pgsql` configured as a NSS provider: this makes all
  users in the DB visible in the `getpwent(3)` functions family, making
  them “be there on the system”;
- having a script set as SSH `AuthorizedKeysCommand` that queries for a
  user's `passwd.data` and pipe it to `jq '.ssh_keys | .[]'`.


#### `libnss-pgsql` configuration

The main part of the configuration of `libnss-pgsql` is to set the queries
used to retrieve information from the database.  Passwords are systematically
set to be `!`: this is a value that cannot possibly match any password hash
in `crypt(3)` format.

Extracting user information is fairly straightforward:

	# Returns (name, passwd, gecos, dir, shell, uid, gid) for a given name or uid, or all
	getpwnam = SELECT name, '!', data->>'name', homedir, data->>'shell', uid, uid FROM passwd WHERE name = $1
	getpwuid = SELECT name, '!', data->>'name', homedir, data->>'shell', uid, uid FROM passwd WHERE id   = $1
	allusers = SELECT name, '!', data->>'name', homedir, data->>'shell', uid, uid FROM passwd


Retrieving group-related data is a bit harder,as  groups are either a user's primary
group, which shares the same name and id, or an auxiliary group described in the
`group` table:

	# Returns (name, passwd, gid) for a given name or gid, or all
	getgrnam  = SELECT name, '!', gid FROM group  WHERE name = $1
	      UNION SELECT name, '!', uid FROM passwd WHERE name = $1
	getgrgid  = SELECT name, '!', gid FROM group  WHERE gid  = $1
	      UNION SELECT name, '!', uid FROM passwd WHERE uid  = $1
	allgroups = SELECT name, '!', gid FROM group
	      UNION SELECT name, '!', uid FROM passwd

Finally, we need a query to link together users and auxiliary groups:

	# Returns all auxiliary group ids a user is a member of
	groups_dyn = SELECT gid FROM passwd JOIN aux_groups USING (uid) WHERE name = $1
	
	# Returns all uids belonging to a given group
	getgroupmembersbygid = SELECT name FROM passwd WHERE uid = $1
	                 UNION SELECT name FROM passwd JOIN aux_groups USING (uid) WHERE gid = $1
```


### `mail.hashbang.sh`

Use directly Postfix's Postgres virtual table support.
