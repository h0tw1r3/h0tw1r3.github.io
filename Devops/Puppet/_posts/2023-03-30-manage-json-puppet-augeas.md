---
layout: post
title: "Manage JSON with Puppet and Augeas"
tags: ["puppet", "json", "augeas"]
---
[JSON] (JavaScript Object Notation) is a popular data interchange format used in many applications. As they become more complex, managing [JSON] configuration files can become a daunting task. Fortunately, automation tools like [Puppet] and [Augeas] can simplify the process.

Lets explore how to use [Puppet] to automate changes to JSON configuration files.

## What is Puppet?

[Puppet] is an open-source configuration management tool that helps system administrators automate the deployment, configuration, and management of their infrastructure. With Puppet, you can define the desired state of your infrastructure using code, and Puppet will automatically enforce that state on your systems.

## What is Augeas?

[Augeas] is a configuration editing tool that provides a simple API for editing configuration files. Augeas is particularly useful for editing configuration files that have complex or non-standard syntax.

## Example Configuration

Let's say you have a JSON configuration file for your web application. In the following examples, we'll be using the following JSON in a file named `/tmp/myapp.conf`.

```json
{
    "database": {
        "host": "localhost",
        "port": 5432,
        "name": "mydb",
        "user": "myuser",
        "password": "mypassword"
    },
    "server": {
        "host": "localhost",
        "port": 8080
    }
}
```

## Simple Example

Suppose you want to change the port number for the "server" section from 8080 to 8000.

1. Create a Puppet manifest file, `/tmp/myapp.pp`, that defines the desired state of the JSON configuration file:

   ```puppet
   augeas { "change_server_port":
     incl    => '/tmp/myapp.conf',
     lens    => 'Json.lns',
     context => '/files/tmp/myapp.conf',
     changes => [
       'set dict/entry[.="server"]/dict/entry[.="port"]/number 8000',
     ],
   }
   ```

2. Apply the Puppet manifest:

   ```sh
   cd /tmp && puppet apply myapp.pp
   ```

   ```output
   Notice: Compiled catalog for test.local in environment production in 0.02 seconds
   Notice: /Stage[main]/Main/Augeas[change_server_port]/returns: executed successfully
   Notice: Applied catalog in 0.02 seconds
   ```

3. Review the JSON configuration file. Notice Puppet changed the value!

   ```json
   {
       "database": {
           "host": "localhost",
           "port": 5432,
           "name": "mydb",
           "user": "myuser",
           "password": "mypassword"
       },
       "server": {
           "host": "localhost",
           "port": 8000
       }
   }
   ```

If you run `puppet apply` again, there should be no changes:

```
$ puppet apply myapp.pp
Notice: Compiled catalog for test.local in environment production in 0.03 seconds
Notice: Applied catalog in 0.02 seconds
```

That's the idempotent power of Puppet!

## Advanced Example

How about managing arbitrary dictionary values from [hiera]?

1. Modify the Puppet manifest file, `/tmp/myapp.pp`:

   ```puppet
   class myapp (
     String $conf_path = '/tmp/myapp.conf',
     Hash   $config    = {},
   ) {
     $changes = $config.map |$kv| {
       $keys = $kv[0].split('/')
   
       Integer[0,$keys.length].reduce([]) |$memo, $i| {
         if $i == $keys.length {
           Integer[0,($memo.length - 1)].map |$mi| {
             $change = sprintf('defnode entry %s "%s"', $memo[$mi][1,-1], $keys[$mi])
             if ($mi + 1) == $memo.length {
               $json_type = $kv[1] ? {
                 Boolean => 'const',
                 Numeric => 'number',
                 default => 'string',
               }
               [ $change, sprintf('set $entry/%s "%s"', $json_type, $kv[1]) ]
             } else {
               [ $change ]
             }
           }
         } else {
           $memo << sprintf('%s/dict/entry[.="%s"]', $memo.join('/'), $keys[$i])
         }
       }
     }.flatten
   
     augeas { 'myapp':
       incl    => $conf_path,
       lens    => 'Json.lns',
       context => "/files${conf_path}",
       changes => $changes,
     }
   }
   
   include myapp
   ```

2. Create a hiera configuration file, `/tmp/hiera.yaml`:

   ```yaml
   ---
   version: 5
   defaults:
     datadir: .
     data_hash: yaml_data
   hierarchy:
    - name: Common
      data_hash: yaml_data
      path: 'myapp.yaml'
   ```

3. Create the hiera data file, `/tmp/myapp.yaml`:

   ```yaml
   ---
   myapp::config:
     database/ssl: true
     database/host: db.somewhere.local
     server/port: 9000
   ```
   
4. Apply the Puppet manifest:

   ```sh
   cd /tmp && puppet apply --hiera_config hiera.yaml myapp.pp
   ```

   ```output
   Notice: Compiled catalog for test.local in environment production in 0.04 seconds
   Notice: /Stage[main]/Myapp/Augeas[myapp]/returns: executed successfully
   Notice: Applied catalog in 0.04 seconds
   ```

5. Review the JSON configuration file. Notice Puppet changed the value!

   ```json
   {
       "database": {
           "host": "db.somewhere.local",
           "port": 5432,
           "name": "mydb",
           "user": "myuser",
           "password": "mypassword"
       ,"ssl":true},
       "server": {
           "host": "localhost",
           "port": 9000
       }
   }
   ```

   _Augeas does not keep the JSON "pretty" when adding new keys._

[JSON]: https://www.json.org/
[Puppet]: https://www.puppet.com/docs/puppet/
[Augeas]: https://forge.puppet.com/modules/puppetlabs/augeas_core/readme
[hiera]: https://www.puppet.com/docs/puppet/latest/hiera.html
