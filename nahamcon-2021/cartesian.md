# Cartesian, Nahamcon CTF 2021

## Context

This challenge was part of the mission path:

```
This is Stage 3 of Path 4 in The Mission. After solving this challenge, you may need to refresh the page to see the newly unlocked challenges.

You don't need to be a "cartographer" to help Gus. But you might need to be a just a "-grapher."

Click the Start button to begin this challenge.
```

(Author: [@Congon4tor](https://twitter.com/congon4tor))

The website is a blog kind of web UI. After playing a bit around I noticed that there's a graphql endpoint at `/graphql`. I didn't know about graphql and had to do my own research. There were quite a few past CTFs playing with this. [This resource](https://jaimelightfoot.com/blog/hack-in-paris-2019-ctf-meet-your-doctor-graphql-challenge/) was full of good pointers.

### Finding the flag

In short, graphql some sort of database-like backend and that endpoint allows querying data via a particular query language.

It turns out that graphql has introspection features, and that on this particular endpoint those features are still enabled. This allows exploring the schema/data. For instance, querying for types via `{__schema { types { name }}}` returns:

```
"data":{"__schema":{"types":[{"name":"__TypeKind"},{"name":"__InputValue"},{"name":"__EnumValue"},{"name":"Query"},{"name":"Post"},{"name":"User"},{"name":"__Schema"},{"name":"__Type"},{"name":"__Directive"},{"name":"__DirectiveLocation"},{"name":"Mutation"},{"name":"Boolean"},{"name":"Int"},{"name":"String"},{"name":"Auth"},{"name":"__Field"}
```

I then looked for existing queries via the following request:

```
{"query":"{__type(name: \"Query\") {name fields { name type { name kind }}}}"}
```

The output reveals there's a `flag` scalar in GraphQL:

```
...
{"name":"flag","type":{"kind":"SCALAR","name":"String"}},
...
```

Trying to run a `{flag}` query should have revealed the contents of that string, however upon execution the backend will respond with an error, telling you that only the admin can access its value. This seems like the right path but we'll have to dig further.

## Can I haz admin password?

I dumped fully the graphql (kudos to https://apis.guru/graphql-voyager/) and explored types and type comments. I noticed a `password` field on `User`, interesting. Since the admin had posted blogs on the instance, I just modified the blog retrieving query that was issued from the main page to also dump the admin password:

```
{
"query":"{ posts {id name image content author {username password  __typename} __typename}}"
}
```

That query leaks out admin "password": `52a9715ba147f750d0751aeccc22b8cc2ff7a9db`. The graphql comments on the User objects hints that this is in fact `sha1(password.salt)`.

Digging further, I then looked into graphql mutations:

```
{"query":"mutation {createUser(username: \"foo\", password:\"bar\"){id username password}}"}
```

Creates a User.

And:

```
{"query":"mutation {authenticateUser(username: \"foo\", password:\"bar\"){token}}"}
```

authenticates the user. Checking what's returned, it's a JWT token. (you present it with `Authentication: Bearer <token>` -- trying to look into that token there was nothing interesting). So I can create my own users and log as these users.

I tried posting a few graphs/blogs, and it did not seem to trigger loading of images or links behind the scenes.

## Summary 

* There's a flag that can be dumped if I present an Admin auth token.
* I know the admin hashed password and its format. I do not know the salt.
* I can create an arbitrary amount of users / passwords
* I can log in as those users
* I can post content as those users but nothing useful seems to happen.

I was stuck after that. Remaining ideas:

* Identify a graphql injection path.
* Get to the salt, then to the original admin password.

**Disclaimer: this was the state I was in at the end of the CTF. I figured the rest after the challenge, checking my solution with the challenge author.**

## Intended solution

1. Create a user, and store `$user_password` and `$user_hash`
1. [hashcat](https://hashcat.net/wiki/doku.php?id=example_hashes) mode 120 is `sha1($salt.$pass)`. In this case we know a password but not a hash, so we can run something like `hashcat -m 120 -a 3` where the input is `$user_hash:$user_password`. This returns you the salt.
1. Run `hashcat -m 110 -a 3` on `$admin_hash:$salt` to figure out the admin password.
1. Login as the user, and run again the `{flag}` query to exfiltrate the flag.

Lack of familiarity with hashcat prevented me from completing the challenge, but I had fun. I'll catch this flag next time ;-)
