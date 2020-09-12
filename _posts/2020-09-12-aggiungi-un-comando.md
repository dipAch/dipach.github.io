---
layout: post
title: solving palindrome
tags: [redis, deep-dive, algorithms]
---


# Wander in Redis Land!
---

There are just a handful of articles that prove helpful when trying to learn the internals of a project like redis, and even fewer that actually demonstrates the internals in a hands-on manner. In order to bridge that gap, here is an attempt to explore and study redis' anatomy while also implementing a (fairly serious) redis command from the ground up to provide a more involved experience to the reader.
It serves as a good baseline for anyone who tends to dive deep into the burrows of redis world and prefers practicality over monotonous textual ingenuity.

## The Goal
---

Let us add a new command that will check whether a given key of type string is a palindrome or not?
Quite exciting, eh?!

## A brief about the important C structs within Redis
---

**client Struct**
---

An important Redis data structure is the one defining a client. The structure has many fields, below are some of the members:

> **redis/src/server.h**

```
...
...

struct client {
	... other fields ...
	connection *conn;
    sds querybuf;
    int argc;
    robj **argv;
    redisDb *db;
    uint64_t flags;
    list *reply;
    char buf[PROTO_REPLY_CHUNK_BYTES];
    ... many other fields ...
} client;

...
...
```

The client structure defines a **Connected Client**:

* The `conn` field is a struct that has the client socket file descriptor as its member field.
* `argc` and `argv` are populated with the command the client is executing, so that functions implementing a given Redis command can read the arguments.
* `querybuf` accumulates the requests from the client, which are parsed by the Redis server according to the Redis protocol and executed by calling the implementations of the commands the client is executing.
* `reply` and `buf` are dynamic and static buffers that accumulate the replies the server sends to the client. These buffers are incrementally written to the socket as soon as the file descriptor is writable.

**redisObject Struct**
---

As you can see in the client structure above, arguments in a command are described as `robj` structures. The following is the full `robj` structure, which defines a **Redis Object**:

> **redis/src/server.h**

```
...
...

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;

...
...
```

Basically this structure can represent all the basic Redis data types like strings, lists, sets, sorted sets and so forth.

The interesting thing is that it has a `type` field, so that it is possible to know what type a given object has, and a `refcount`, so that the same object can be referenced in multiple places without allocating it multiple times.

Finally the `ptr` field points to the actual representation of the object, which might vary even for the same type, depending on the `encoding` used.

# Registering a new command
---

All redis commands are defined inside `redis/src/server.c` in an array of structs called the **redisCommandTable**. We will add our new command entry inside this command table array like so,

> **redis/src/server.c**

```
...
...

struct redisCommand redisCommandTable[] = {
    ...
    ...

	{"palindrome", palindromeCommand, 2,
     "read-only fast @string",
     0, NULL, 1, 1, 1, 0, 0, 0},

    ...
    ...
};

...
...
```

The global variable **redisCommandTable** defines all the Redis commands, specifying the name of the command, the function implementing the command (a.k.a the handler), the number of arguments required, and other properties of each command.

In the above example `2` is the number of arguments the command takes, while `"read-only fast @string"` are the command flags as documented in the command table top comment inside `redis/src/server.c`.

**Anatomy of the command struct**
---

* name:            A string representing the command name.
				   ==> "palindrome"
* function:        Pointer to the C function implementing the command.
				   ==> palindromeCommand
* arity:           Number of arguments, it is possible to use -N to say >= N.
				   ==> 2
* sflags:          Command flags as string. See below for a table of flags.
				   ==> "read-only fast @string"
* flags:           Flags as bitmask. Computed by Redis using the 'sflags' field.
				   ==> 0
* get_keys_proc:   An optional function to get key arguments from a command.
                   This is only used when the following three fields are not
                   enough to specify what arguments are keys.
                   ==> NULL
* first_key_index: First argument that is a key
				   ==> 1
* last_key_index:  Last argument that is a key
				   ==> 1
* key_step:        Step to get all the keys from first to last argument.
                   For instance in MSET the step is two since arguments
                   are key,val,key,val,...
                   ==> 1
* microseconds:    Microseconds of total execution time for this command.
				   ==> 0
* calls:           Total number of calls of this command.
				   ==> 0
* id:              Command bit identifier for ACLs or other goals.
				   ==> 0

* The **flags, microseconds and calls fields** are computed by Redis and should always
  be set to zero (0).


* Command flags are expressed using space separated strings, that are turned
* into actual flags by the populateCommandTable() function.

**Below are the meaning of the flags, that we have used:**
---

* read-only:   All the non special commands just reading from keys without
               changing the content, or returning other informations like
               the TIME command. Special commands such administrative commands
               or transaction related commands (multi, exec, discard, ...)
               are not flagged as read-only commands, since they affect the
               server or the connection in other ways.
* fast:        Fast command: O(1) or O(log(N)) command that should never
               delay its execution as long as the kernel scheduler is giving
               us time. Note that commands that may trigger a DEL as a side
               effect (like SET) are not fast commands.

* The following additional flags are only used in order to put commands
  in a specific ACL category. Commands can have multiple ACL categories.
 
  @keyspace, @read, @write, @set, @sortedset, @list, @hash, **@string**, @bitmap,
  @hyperloglog, @stream, @admin, @fast, @slow, @pubsub, @blocking, @dangerous,
  @connection, @transaction, @scripting, @geo.

  In our new command, we are using the **@string** ACL flag.
 
* The read-only flag implies the @read ACL category.
* The fast flag implies the @fast ACL category.

The C struct that constitutes a redis command is extracted below. The below members map to the above field
descriptions and is easily comprehensible (assisted by code comments).

> **redis/src/server.h**

```
...
...

struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags;   /* Flags as string representation, one char per flag. */
    uint64_t flags; /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls;
    int id;     /* Command ID. This is a progressive ID starting from 0 that
                   is assigned at runtime, and is used in order to check
                   ACLs. A connection is able to execute a given command if
                   the user associated to the connection has this command
                   bit set in the bitmap of allowed commands. */
};

...
...
```

After the command operates in some way, it returns a reply to the client, usually using `addReply()` or a
similar function defined inside **redis/src/networking.c**. We will see this in action as we make progress.

# Defining our palindrome command handler
---

Redis commands handlers (entry-points) are defined in the following way:

> **redis/src/t_string.c**

```
...
...

void palindromeCommand(client *c) {
    robj *o;
    if ((o = lookupKeyReadOrReply(c, c->argv[1], shared.null[c->resp])) == NULL ||
        checkType(c, o, OBJ_STRING)) return;
    stringObjectPalindrome(c, o);
}

...
...
```

We take in a connected client instance. Then we check if the key exists or not.
If it exists, we get the value for the key and store it as a **redisObject** instance, else we send a **NULL** reply to the client (which is done for non-existant keys) from within the **lookupKeyReadOrReply** function itself. For more details, check out the **redis/src/db.c::lookupKeyReadOrReply** function.

```
if ((o = lookupKeyReadOrReply(c, c->argv[1], shared.null[c->resp])) == NULL || ...) return;


An Example Case:
---------------

127.0.0.1:6379> PALINDROME ghost
(nil)
127.0.0.1:6379>
```

If the key exists, it's well and good but we also need to check if we are dealing with the correct type of the object. Now, if we are expecting a string and instead retrieve a **redisDict** (HashMap) object, we should definitely not proceed. So that forms the second sanity in our **OR** conditional check. For additional details, check out the **redis/src/object.c::checkType** function.

```
if ( ... || checkType(c, o, OBJ_STRING)) return;


An Example Case:
---------------

127.0.0.1:6379> HMSET myhash field1 "Hello" field2 "World"
OK
127.0.0.1:6379> HGET myhash field1
"Hello"
127.0.0.1:6379> PALINDROME myhash
(error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379>
```

Any new function that we introduce, needs the function signature (or prototype) defined in the `redis/src/server.h` header file. So lets add the the command handler function's prototype there.

> **redis/src/server.h**

```
...
...
void palindromeCommand(client *c);
...
...
```

# Project Palindrome
---

Now, let's get to the heart of the topic. The command handler needs to call the below function **stringObjectPalindrome** to check if the string value is a palindrome or not?

Again, we would need to add the function signature to the `redis/src/server.h` header file like before. So, let's get that out of the way first.

> **redis/src/server.h**

```
...
...
void stringObjectPalindrome(client *c, robj *o);
...
...
```

Inside `redis/src/object.c`, there are all the functions that operate with Redis objects at a basic level, like functions to allocate new objects, handle the reference
counting and so forth.

Below file is where we would want to define the core logic of our brand new command.

> **redis/src/object.c**

```
...
...

void stringObjectPalindrome(client *c, robj *o) {
    serverAssertWithInfo(NULL, o, o->type == OBJ_STRING);

    sds value;
    size_t string_length, tail_pointer;

    if (sdsEncodedObject(o)) {
        value = o->ptr;
        string_length = sdslen(value);
    } else {
        char buf[64];
        string_length = ll2string(buf, sizeof(buf), (long long)o->ptr);
        value = (sds)buf;
    }

    tail_pointer = string_length - 1;

    for (size_t idx=0; idx < string_length; idx++) {
        if (value[idx] != value[tail_pointer]) {
            addReplyBulkCString(c, "false");
            return;
        }
        tail_pointer--;
    }

    addReplyBulkCString(c, "true");
}

...
...
```

The function definition takes in an **robj** instance (an **SDS** object that represents a string in the redis world). We also need the **client** object as we need to send back the reply to the connected session.

**What's happening here?!**
---

* First we have an assert call that checks whether we are dealing with the correct data type. In our case, we see if the object type is actually a string. Nothing fancy here.
* Next up, we declare a bunch of variables that would be needed when checking for a palindrome.
* `sds` is the Redis string wrapper over normal c strings. It is a struct that has other members defined that provides much efficiency at run-time. An sds is of type char *
* The next if / else block checks if we the string is encoded as an `sds` object or not. The else part deals with converting the object to an sds encoded string. String values that represent numeric entities need to be transformed to the correct sds encoded data type. We do this using the function **ll2string**. This function also returns the length of the newly converted string object and type casts it into an `sds` object. For additional details, check out the **redis/src/util.c::ll2string** function in the redis source code.
* If the object is already `sds` encoded, we can directly find the length of the string using the `sdslen` library function.
* What follows next is the typical palindrome check logic. Check the characters in each end of the string, one iteration at a time.
* Now the important part here is replying back to the client that issued the command. For this, we use one of the many variants for replying namely, **redis/src/networking.c::addReplyBulkCString**. The usage is quite straightforward. We pass the client object that was provided as an argument to the calling function and a C string literal. Thats it, we have added a new redis command 👏!

# Are we done?
---

Nothing is done actualy without writing some tests. And that's a fact! So, out experiment should be no less.

Let's put together a few tests, that verify the behaviour of our newly added redis command. Redis tests are written in [Tcl (Tool Command Language)](https://github.com/redis/redis).

> **redis/tests/unit/type/string.tcl**

```
	...
	...
	...

	test "PALINDROME against integer-encoded value" {
        r set myinteger -9900
        assert_equal "false" [r palindrome myinteger]
        r set myinteger 1001
        assert_equal "true" [r palindrome myinteger]
        r set myinteger 999
        assert_equal "true" [r palindrome myinteger]
    }

    test "PALINDROME against even length plain string" {
        r set mystring "foozzz"
        assert_equal "false" [r palindrome mystring]
        r set mystring "foooof"
        assert_equal "true" [r palindrome mystring]
    }

    test "PALINDROME against odd length plain string" {
        r set mystring "adcbz"
        assert_equal "false" [r palindrome mystring]
        r set mystring "abcba"
        assert_equal "true" [r palindrome mystring]
    }

    ...
    ...
    ...
```

With that done, let's see if we have everything working (as expected)!

# Let's take it out for a spin!
---

Make sure you're inside your docker container's, redis root directory. Everything we will do is from the **REDIS_HOME**.

```
$ docker exec -it project-root_go-app_1 bash
bash-5.0#
bash-5.0# cd /webapps/redis/
bash-5.0# pwd
/webapps/redis
bash-5.0#
```

If you've compiled redis from source before hand, it is a good idea to cleanup the previous build and start fresh. Run below **make** directive to do so,

```
bash-5.0#
bash-5.0# make distclean
cd src && make distclean
make[1]: Entering directory '/webapps/redis/src'
...
...
...
bash-5.0#
bash-5.0#
```

Build redis using the **make** directive with **CFLAGS** option set.

```
bash-5.0#
bash-5.0# pwd
/webapps/redis
bash-5.0# 
bash-5.0# make CFLAGS="-ggdb -Og -g3 -fno-omit-frame-pointer"
cd src && make all
make[1]: Entering directory '/webapps/redis/src'
    CC Makefile.dep
	...
	...
	...
	...
    LINK redis-server
    INSTALL redis-sentinel
    CC redis-cli.o
    LINK redis-cli
    CC redis-benchmark.o
    LINK redis-benchmark
    INSTALL redis-check-rdb
    INSTALL redis-check-aof

Hint: It's a good idea to run 'make test' ;)

make[1]: Leaving directory '/webapps/redis/src'
bash-5.0#
bash-5.0#
```

It's also a good idea to run the tests (post running **make**), and verify that all as well as our newly added tests are passing.

```
bash-5.0# 
bash-5.0# make test
cd src && make test
make[1]: Entering directory '/webapps/redis/src'
    CC Makefile.dep
Cleanup: may take some time... OK
Starting test server at port 11114
...
...
...
[ok]: PALINDROME against integer-encoded value
[ok]: PALINDROME against even length plain string
[ok]: PALINDROME against odd length plain string
...
...
...
\o/ All tests passed without errors!

Cleanup: may take some time... OK
make[1]: Leaving directory '/webapps/redis/src'
bash-5.0#
bash-5.0#
```

**Test Drive:**
---

```
$ docker exec -it project-root_go-app_1 bash
bash-5.0#
bash-5.0# cd /webapps/redis/
bash-5.0# pwd
/webapps/redis
bash-5.0#
bash-5.0# ./src/redis-cli
127.0.0.1:6379> set redis baba
OK
127.0.0.1:6379> PALINDROME redis
"false"
127.0.0.1:6379>
127.0.0.1:6379> set redis abba
OK
127.0.0.1:6379> PALINDROME redis
"true"
127.0.0.1:6379>
127.0.0.1:6379> set redis:odd:length cdzxc
OK
127.0.0.1:6379> PALINDROME redis:odd:length
"false"
127.0.0.1:6379>
127.0.0.1:6379> set redis:odd:length cdzdc
OK
127.0.0.1:6379> PALINDROME redis:odd:length
"true"
127.0.0.1:6379>
127.0.0.1:6379> set numeric 1001
OK
127.0.0.1:6379> PALINDROME numeric
"true"
127.0.0.1:6379> set numeric 991
OK
127.0.0.1:6379> PALINDROME numeric
"false"
127.0.0.1:6379>
```

## Recommended Good Reads
---

* [Redis Internals: Server Initializaton, SDS, Lists, Dicts (HashMaps), Eventloop](https://hellokangning.github.io/post/)
* [Tracing a Redis GET & SET](https://www.pauladamsmith.com/blog/2011/03/redis_get_set.html)
* [Though old, but a redis internals classic](https://www.pauladamsmith.com/articles/redis-under-the-hood.html)

## Other String Command Ideas:

* Command to find the Longest Palindromic Substring in a string.
* Command to retrieve the Longest Substring Without Duplication from a string.

Other programming problems that deal with other data structures are also ideal candidates.

Redis being a datastructure server, serves as a good platform to apply DS & Algo problems. Also, you become
familiar with the internals of a widely popular open-source software. I would say, "It's a win-win".

# Outro
---

Well, that's it folks for this post. Thanks for your time and attention. See you the next time around!
