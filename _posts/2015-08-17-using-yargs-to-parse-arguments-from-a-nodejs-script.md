---
layout:      post
title:       Using Yargs to parse Arguments from a Node.js Script
description: 
headline:    "Using Yargs to parse Arguments from a Node.js Script"
categories:  [Node.js]
tags:        [Node.js,Yargs]
image:       
comments:    false
mathjax:     
featured:    false
published:   true
---


A few weeks ago I was writing (another version of) a User Sync script in Node.js and I thought it would be nice if I could make most of parameters configurable so that I could control the script flow from the console, without having to change the actual content of the file. After spending some time looking for the different options available and comparing them, [Yargs](https://github.com/bcoe/yargs) was the preferred choice because it is both simple and powerful. 

In Yargs, each argument has a unique name (the identifier or hash) and this name can be a single character (hyphenated) or multiple letters (double hyphenated). All the arguments can have an alias (long version of the name), a description and a default value. Yargs also supports boolean, long and even grouped arguments! There is nothing like a real example to understand what this is all about:

```javascript
var argv = require('yargs')  
  .usage('Usage: node $0 [options]')  
  .demand('f').alias('f', 'file-name').nargs('f', 1).describe('f', 'The CSV Source File Name').default('f', 'users.csv')  
  .demand('b').alias('b', 'instance-url').nargs('b', 1).describe('b', 'The Base URL for the Instance').default('b', 'https://customer.jiveon.com')  
  .demand('u').alias('u', 'admin-username').nargs('u', 1).describe('u', 'The Admin Username').default('u', 'admin')  
  .demand('p').alias('p', 'admin-password').nargs('p', 1).describe('p', 'The Admin Password')  
  .demand('d').alias('d', 'date-format').nargs('d', 1).describe('u', 'The Date Format on the Instance').default('d', 'DD/MM/YYYY')  
  .demand('s').alias('s', 'max-sockets').nargs('s', 1).describe('s', 'The maximum number of Sockets available for HTTP(S) requests').default('s', 200)  
  .demand('r').alias('r', 'max-retries').nargs('r', 1).describe('r', 'The maximum number of Retries when there is an Error calling the API').default('r', 5)  
  .demand('w').alias('w', 'sleep-seconds').nargs('w', 1).describe('w', 'The number of Seconds to sleep between each request to the API').default('w', 10)  
  .boolean('pn').alias('pn', 'phone-numbers').describe('pn', 'Enables the Update of Phone Numbers for the Users').default('pn', true)  
  .boolean('um').alias('um', 'update-managers').describe('um', 'Enables the Update of Managers for the Users').default('um', true)  
  .boolean('up').alias('up', 'user-provisioning').describe('up', 'Enables the Provisioning of Users when no record is Found').default('up', false)  
  .boolean('fu').alias('fu', 'force-update').describe('fu', 'Forces the Update of the User when there are no changes in the CSV').default('fu', false)  
  .boolean('sp').alias('sp', 'show-progress').describe('sp', 'Shows the Load and Update progress for each interaction with the API').default('sp', false)  
  .help('h').alias('h', 'help')    
  .argv;  
```

How to get the actual content of the arguments? Just referrer to the unique name of the argument as shown here:

```javascript
var fileName = argv.f;  
var instanceUrl = argv.b;   
var adminUsername = argv.u  
var adminPassword = argv.p;  
var dateFormat = argv.d;  
var maxSockets = argv.s;  
var maxRetries = argv.r;  
var sleepSeconds = argv.w;  
var phoneNumbersEnabled = argv.pn;  
var userManagersEnabled = argv.um;  
var userProvisioningEnabled = argv.up;  
var forceUpdate = argv.fu;  
var showProgressEnabled = argv.sp;  
```

Yes, this is useful for the person writing the script but how does this helps the person running the script? Because once the script has been configured to use Yargs, it can now generate UNIX-like help on how to execute, including all the arguments available and their meaning. Below is the example help output which is the result of the configuration entered above:

```javascript
Usage: node users.js [options]  
    
    
Options:  
  -f, --file-name            The CSV Source File Name  
                                               [required] [default: "users.csv"]  
  -b, --instance-url         The Base URL for the Instance  
                             [required] [default: "https://customer.jiveon.com"]  
  -u, --admin-username       The Date Format on the Instance  
                                                   [required] [default: "admin"]  
  -p, --admin-password       The Admin Password                       [required]  
  -s, --max-sockets          The maximum number of Sockets available for HTTP(S  
                             ) requests                [required] [default: 200]  
  -r, --max-retries          The maximum number of Retries when there is an  
                             Error calling the API       [required] [default: 5]  
  -w, --sleep-seconds        The number of Seconds to sleep between each request  
                             to the API                 [required] [default: 10]  
  --pn, --phone-numbers      Enables the Update of Phone Numbers for the Users  
                                                       [boolean] [default: true]  
  --um, --update-managers    Enables the Update of Managers for the Users  
                                                       [boolean] [default: true]  
  --up, --user-provisioning  Enables the Provisioning of Users when no record is  
                             Found                    [boolean] [default: false]  
  --fu, --force-update       Forces the Update of the User when there are no  
                             changes in the CSV       [boolean] [default: false]  
  --sp, --show-progress      Shows the Load and Update progress for each  
                             interaction with the API  
                                                      [boolean] [default: false]  
  -h, --help                 Show help                                 [boolean]  
  -d, --date-format                           [required] [default: "DD/MM/YYYY"]  
```

You can do a few more things with Yargs that I have not explored like multiple values for the same argument, localized help messages and several other advanced options. Hope this helps someone!