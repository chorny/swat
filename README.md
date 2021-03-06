# SYNOPSIS

SWAT is Simple Web Application Test ( Tool )

    $  swat examples/google/ google.ru
    /home/vagrant/.swat/reports/google.ru/00.t ..
    # start swat for google.ru/
    # try num 2
    ok 1 - successful response from GET google.ru/
    # data file: /home/vagrant/.swat/reports/google.ru/content.GET.txt
    ok 2 - GET / returns 200 OK
    ok 3 - GET / returns Google
    1..3
    ok
    All tests successful.
    Files=1, Tests=3, 12 wallclock secs ( 0.00 usr  0.00 sys +  0.02 cusr  0.00 csys =  0.02 CPU)
    Result: PASS

# WHY

I know there are a lot of test tools and frameworks, but let me  briefly tell _why_ I created swat.
As devops, I update dozens of web application weekly, sometimes I just have _no time_ to sit and wait, 
while dev guys or QA team ensure that deployment is fine and nothing breaks on the road. 
So I need a **tool to run smoke tests against web applications**. 
Not just a tool, but the way to **create such tests from scratch in a way that's easy and fast enough**. 

So this is how I came up with the idea of swat. 

# Key features

SWAT:

- is a very pragmatic tool, designed for the job to be done in a fast and simple way
- has simple and yet flexible DSL with low price mastering ( see my tutorial )
- produces [TAP](https://testanything.org/) output
- leverages famous [perl prove](http://search.cpan.org/perldoc?prove) and [curl](http://curl.haxx.se/) utilities

# Install

Swat relies on curl utility to make http requests. Thus first you need to install curl:

    $ sudo apt-get install curl

Also swat client is a bash script so you need bash. 

Then you install swat cpan module:

    sudo cpan install swat

## Install from source

    # useful for contributors
    perl Makefile.PL
    make
    make test
    make install

# Swat mini tutorial

For those who love to make long story short ...

## Create tests

    mkdir  my-app/ # create a project root directory to contain tests

    # define http URIs application should response to

    mkdir -p my-app/hello # GET /hello
    mkdir -p my-app/hello/world # GET /hello/world

    # define the content to return by URIs

    echo 200 OK >> my-app/hello/get.txt
    echo 200 OK >> my-app/hello/world/get.txt

    echo 'This is hello' >> my-app/hello/get.txt
    echo 'This is hello world' >> my-app/hello/world/get.txt

## Run tests

    swat ./my-app http://127.0.0.1

# DSL

Swat DSL consists of 2 parts. Routes and Swat Data.

## Routes

Routes are http resources a tested web application should have.

Swat utilize file system to get know about routes. Let we have a following project layout:

    example/my-app/
    example/my-app/hello/
    example/my-app/hello/get.txt
    example/my-app/hello/world/get.txt

When you give swat a run

    swat example/my-app 127.0.0.1

It will find all the _directories with get.txt or post.txt or put.txt files inside_ and "create" routes:

    GET hello/
    GET hello/world

It is possible to run a subset of swat tests using a `test_file` variable:

    # run a single `hello/world/get.txt' test
    test_file=example/my-app/hello/world/get.txt swat example/my-app 127.0.0.1

    # the same as above but in less precise maner:
    test_file=example/my-app/hello/world/ swat example/my-app 127.0.0.1

    # run all `hello' tests:
    test_file=example/my-app/hello/ swat example/my-app 127.0.0.1

Ok, when we are done with routes we need to setup swat data.

## Swat data

Swat data is DSL to describe/generate validation checks you apply to content returned from web application.

Swat data is stored in swat data files, named get.txt or post.txt or put.txt. 

The validation process looks like:

- Swat recursively find files named **get.txt** or **post.txt** or **put.txt** in the project root directory to get swat data.
- Swat parse swat data file and _execute_ entries found. At the end of this process swat creates a _final check list_ with 
["Check Expressions"](#check-expressions).
- For every route swat makes http requests to web application and store content into text file 
- Every line of text file is validated by every item in a _final check list_

_Objects_ found in test data file are called _swat entries_. There are _3 basic type_ of swat entries:

- Check Expressions
- Comments
- Perl Expressions and Generators

### Check Expressions

This is most usable type of entries you  may define at swat data file. _It's just a string should be returned_ when swat request a given URI. Here are examples:

    200 OK
    Hello World
    <head><title>Hello World</title></head>

Using regexps.

Regexps are check expressions with the usage of &lt;perl regular expressions> instead of plain strings checks.
Everything started with `regexp:` marker would be treated as perl regular expression.

    # this is example of regexp check
    regexp: App Version Number: (\d+\.\d+\.\d+)

### Comments

Comments entries are lines started with `#` symbol, swat will ignore comments when parse swat data file. Here are examples.

    # this http status is expected
    200 OK
    Hello World # this string should be in the response
    <head><title>Hello World</title></head> # and it should be proper html code

### Blank lines

Blank lines found swat data files are ignored. You may use any of them just to improve readability:

    # check http header
    200 OK
    # then 2 blank lines


    # then another check
    HELLO WORLD
    

... But you **can't ignore** blank lines in a `text block matching` context ( see next section ), use `:blank_line` marker to match blank lines:

       # :blank_line marker matches blank lines
       # this is especially useful 
       # when match in text blocks context:

       begin:
           this line followed by 2 blank lines
           :blank_line
           :blank_line
       end:
    

### Matching text blocks

Sometimes it is very helpful match a content _not against a single string_, but against a `block of strings` goes consequentially, like here:

    # this text block
    # consists of 5 strings 
    # goes consequentially 
    # line by line:

    begin:
        # plain strings
        this string followed by
        that string followed by
        another one
        # regexps patterns:
    regexp: with (this|that)
        # and the last one in a block
        at the very end
    end: 

This test will pass when running against this chunk:

    this string followed by
    that string followed by
    another one string
    with that string
    at the very end.

But **won't** pass for this chunk:

    that string followed by
    this string followed by
    another one string
    with that string
    at the very end.

`begin:` `end:` markers decorate \`text blocks\` content. Markers should not be followed by any text at the same line.

Also be aware if you leave "dangling" begin: marker without closing end: somewhere else this will
result in \`text block\` mode till the end of your test, which is probably not you want:

    begin:
    here we begin
    and till the very end of test
    we are in `text block` mode    

### Perl Expressions

Perl expressions are just a pieces of perl code to _get evaled_ by swat when parsing test data files.

Everything started with `code:` marker would be treated by swat as perl code to execute.
There are a _lot of possibilities_! Please follow [Test::More](https://metacpan.org/pod/search.cpan.org#perldoc-Test::More) documentation to get more info about useful function you may call here.

    code: skip('next test is skipped',1) # skip next check forever
    HELLO WORLD


    code: skip('next test is skipped',1) unless $ENV{'debug'} == 1  # conditionally skip this check
    HELLO SWAT

### Generators

Swat entries generators is the way to _create new swat entries on the fly_. Technically speaking it's just a perl code which should return an array reference:
Generators are very close to perl expressions ( generators code is also get evaled ) with major difference:

Value returned from generator's code should be  array reference. The array is passed back to swat parser so it can create new swat entries from it. 

Generators entries start with `:generator` marker. Here is example:

    generator: [ qw{ foo bar baz } ]

This generator will generate 3 swat entries:

    foo
    bar
    baz

As you can guess an array returned by generator should contain _perl strings_ representing swat entries, here is another example:
with generator producing still 3 swat entities 'foo', 'bar', 'baz' :

    generator: my %d = { 'foo' => 'foo value', 'bar' => 'bar value' }; [ map  { ( "# $_", "$data{$_}" )  } keys %d  ] 

This generator will generate 3 swat entities:

    # foo
    foo value
    # bar
    bar value

There is no limit for you! Use any code you want with only requirement - it should return array reference. 
What about to validate web application content with sqlite database entries?

    generator:                                                          \
    
    use DBI;                                                            \
    my $dbh = DBI->connect("dbi:SQLite:dbname=t/data/test.db","","");   \
    my $sth = $dbh->prepare("SELECT name from users");                  \
    $sth->execute();                                                    \
    my $results = $sth->fetchall_arrayref;                              \
    
    [ map { $_->[0] } @${results} ]

As an example take a loot at examples/swat-generators-sqlite3 project

### Multiline expressions

Sometimes code looks more readable when you split it on separate chunks. When swat parser meets  `\` symbols it postpone entry execution and
add next line to buffer. This is repeated till no `\` found on next. Finally swat execute _"accumulated"_ swat entity.

Here are some examples:

    # this is a generator
    # with multiline code
    generator:                  \
    my %d = {                   \
        'foo' => 'foo value',   \
        'bar' => 'bar value',   \
        'baz' => 'baz value'    \
    };                          \
    [                                               \
        map  { ( "# $_", "$data{$_}" )  } keys %d   \
    ]                                               \

    # this is also a generator
    # with a code consists of many lines
    generator: [            \
            map {           \
            uc($_)          \
        } qw( foo bar baz ) \
    ]

    code:                                                       \
    if $ENV{'debug'} == 1  { # conditionally skip this check    \
        skip('next test is skipped',1)                          \ 
    } 
    HELLO SWAT

Multiline expressions are only allowable for perl expressions and generators.

### Generators and Perl Expressions Scope

Swat uses _perl string eval_ when process generators and perl expressions code, be aware of this. 
Follow [http://perldoc.perl.org/functions/eval.html](http://perldoc.perl.org/functions/eval.html) to get more on this.

### PERL5LIB

Swat adds **$project\_root\_directory/lib** to PERL5LIB , so this is convenient convenient to place here custom perl modules:

    example/my-app/lib/Foo/Bar/Baz.pm

Take a look at examples/swat-generators-with-lib/ for working example

### Captures

Captures are pieces of data get captured when swat match content against check expressions using _regexps_:

    # here is content returned.
    # it is just my family ages.
    alex    38
    julia   25
    jan     2
    
    
    # here is swat data

    regexp: /(\w+)\s+(\d+)/

A check expression above will result in a captured data as perl array reference:

    [
        ['alex',    38 ]
        ['julia',   32 ]
        ['jan',     2  ]
    ]

Now captures might be accessed by code generators to get some extra checks:

    code:                               \
    my $total=0;                        \
    for my $c (@{captures()}) {         \
        $total+=$c->[0];                \         
    }                                   \
    cmp_ok( $total,'==',72,"total age of my family" );
    

Thus perl expressions and code generators access captures data calling `captures()` function.

Captures() returns an array reference holding all data captured during _latest regexp check_.

Here some more examples:

    # check if output contains numbers
    # calculate total amount 
    # it should be greater then ten

    regexp: (\d+)
    code:                               \
    my $total=0;                        \
    for my $c (@{captures()}) {         \
        $total+=$c->[0];                \         
    }                                   \
    cmp_ok( $total,'>',10,"total amount is greater than 10" );


    # check if output contains lines 
    # with date formatted as `date: YYYY-MM-DD`
    # check if first date found is yesterday

    regexp: date: (\d\d\d\d)-(\d\d)-(\d\d)
    code:                               \
    use DateTime;                       \
    my $c = captures()->[0];            \
    my $dt = DateTime->new( year => $c->[0], month => $c->[1], day => $c->[2]  ); \
    my $yesterday = DateTime->now->subtract( days =>  1 );     \
    cmp_ok( DateTime->compare($dt, $yesterday),'==',0,"first day found is - $dt and this is a yesterday" );

You also may use `capture()` function to get a _first element_ of captures array:

    # check if output contains numbers
    # a first number should be greater then ten

    regexp: (\d+)
    code: cmp_ok( capture()->[0],'>',10,"first number is greater than 10" );

# Anatomy of swat 

Once swat runs it goes through some steps to get job done. Here is description of such a steps executed in orders

## Run iterator over swat data files

Swat iterator look for all files named get.txt or post.txt or put.txt under project root directory. Actually this is simple bash find loop.

## Parse swat data file

For every swat data file find by iterator parsing process starts. Swat parse data file line by line, at the end of such a process
_a list of Test::More asserts_ is generated. Finally asserts list and other input parameters are serialized as Test::More test scenario 
written into into proper \*.t file.

## Give it a run by prove

Once swat finish parsing all the swat data files there is a whole bunch of \*.t files kept under a designated  temporary directory,
thus every swat route maps into Test::More test file with the list of asserts. Now all is ready for prove run. Internally \`prove -r \`
command is issued to run tests and generate TAP report. That is it.

Below is example how this looks like

### project structure

    $ tree examples/anatomy/
    examples/anatomy/
    |----FOO
    |-----|----BARs
    |           |---- post.txt
    |--- FOOs
          |--- get.txt

    3 directories, 2 files

### swat data files

    # /FOOs 
    FOO
    FOO2
    generator: | %w{ FOO3 FOO4 }|

    # /FOO/BARs
    BAR
    BAR2
    generator: | %w{ BAR3 BAR4 }|
    code: skip('skip next 2 tests',2);
    BAR5
    BAR6
    BAR7

### Test::More Asserts list

    # /FOOs/0.t
    SKIP {
        ok($status, "successful response from GET $host/FOOs") 
        ok($status, "GET /FOOs returns FOO")
        ok($status, "GET /FOOs returns FOO2")
        ok($status, "GET /FOOs returns FOO3")
        ok($status, "GET /FOOs returns FOO4")
    }

    # /FOO/BARs0.t
    SKIP {
        ok($status, "successful response from POST $host/FOO/BARs") 
        ok($status, "POST /FOO/BARs returns BAR")
        ok($status, "POST /FOO/BARs returns BAR")
        ok($status, "POST /FOO/BARs returns BAR3")
        ok($status, "POST /FOO/BARs returns BAR4")
        skip('skip next 2 tests',2);
        ok($status, "POST /FOO/BARs returns BAR5")
        ok($status, "POST /FOO/BARs returns BAR6")
        ok($status, "POST /FOO/BARs returns BAR7")
    }

# POST/PUT requests

Name swat data file as post.txt (put.txt) to make http POST (PUT) requests.

    echo 200 OK >> my-app/hello/post.txt
    echo 200 OK >> my-app/hello/world/post.txt

You may use curl\_params setting ( follow ["Swat Settings"](#swat-settings) section for details ) to define post data, there are some examples:

- `-d` - Post data sending by html form submit.

         # Place this in swat.ini file or sets as env variable:
         curl_params='-d name=daniel -d skill=lousy'

- `--data-binary` - Post data sending as is.

         # Place this in swat.ini file or sets as env variable:
         curl_params=`echo -E "--data-binary '{\"name\":\"alex\",\"last_name\":\"melezhik\"}'"`
         curl_params="${curl_params} -H 'Content-Type: application/json'"

# Swat Settings

Swat comes with settings defined in two contexts:

- Environment variables ( session settings )
- swat.ini files ( home directory , project based, route based and custom settings  )

## Environment variables

Following variables define a proper swat settings.

- `debug` - set to `1,2` if you want to see some debug information in output, default value is `0`
- `debug_bytes` - number of bytes of http response  to be dumped out when debug is on. default value is `500`
- `swat_debug` - run swat in debug mode, default value is `0`
- `ignore_http_err` - ignore http errors, if this parameters is off (set to `1`) returned  _error http codes_ will not result in test fails, 
useful when one need to test something with response differ from  2\*\*,3\*\* http codes. Default value is `0`
- `try_num` - number of http requests  attempts before give it up ( useless for resources with slow response  ), default value is `2`
- `curl_params` - additional curl parameters being add to http requests, default value is `""`, follow curl documentation for variety of values for this
- `curl_connect_timeout` - follow curl documentation
- `curl_max_time` - follow curl documentation
- `port`  - http port of tested host, default value is `80`
- `prove_options` - prove options, default value is `-v`

## Swat.ini files

Swat checks files named `swat.ini` in the following directories

- **~/swat.ini** - home directory settings
- **$project\_root\_directory/swat.ini** -  project based settings 
- **$route\_directory/swat.ini** - route based settings 
- **$cwd/swat.my** - custom settings 

Here are examples of locations of swat.ini files:

     ~/swat.ini # home directory settings 
     my-app/swat.ini # project based settings
     my-app/hello/get.txt
     my-app/hello/swat.ini # route based settings ( route hello )
     my-app/hello/world/get.txt
     my-app/hello/world/swat.ini # route based settings ( route hello/world )

Once file exists at any location swat simply **bash sources it** to apply settings.

Thus swat.ini file should be bash file with swat variables definitions. Here is example:

    # the content of swat.ini file:
    curl_params="-H 'Content-Type: text/html'"
    debug=1
    try_num=3

## Settings priority table

This table describes all the settings with priority levels, the settings with higher priority are applied after settings
with lower priority.

    | context                 | location                   | settings type        | priority  level |
    | ------------------------|--------------------------- | -------------------- | --------------- |
    | swat.ini file           | ~/swat.ini                 | home directory       |       1         |
    | swat.ini file           | project root directory     | project based        |       2         |
    | swat.my  file           | current working directory  | custom settings      |       3         |
    | swat.ini file           | route directory            | route based          |       4         |
    | environment variables   | ---                        | session              |       5         |

# Settings merge algorithm

Thus swat applies settings in order for every route:

- Home directory settings are applied if exist.
- Project based settings are applied if exist.
- Custom settings are applied if exist.
- Route based settings are applied if exist.
- And finally environment settings are applied if exist.

## Custom Settings

Custom settings are way to customize settings for existed swat package. This file should be located at current working directory,
where you run swat from. For example:

    # override http port
    $ echo port=8080 > swat.my
    $ swat swat::nginx 127.0.0.1

Follow section ["Swat Packages"](#swat-packages) to get more about portable swat tests.

# Hooks

Hooks are extension points you may implement to hack into swat compile / runtime workflow. There are two types of hooks:

- Perl hooks
- Bash Hooks

## Perl hooks

Perl hooks are files with perl code \`required\` _in the beginning/end of a swat test_. There are four types of perl hooks:

- **project based perl startup hook**

    File located at `$project_root_directory/hook.pm`. 

    Project based startup hooks are \`required\` _in the beginning_ of a swat test and applied for every route in project 
    and thus could be used for _project initialization_ procedures. 

    For example one could define common generators here:

        # place this in hook.pm file:
        sub list1 { | %w{ foo bar baz } | }
        sub list2 { | %w{ red green blue } | }


        # now we could use it in swat data file
        generator:  list() 
        generator:  list2()    

- **project based perl cleanup hook**

    File located at `$project_root_directory/cleanup.pm`. 

    This hooks is similar to startup hook but \`required\` _in the end_ of a swat test.

- **route based perl startup hooks**

    Files located at `$route_directory/hook.pm`. 

    Routes based startup hooks are applied for every route in project and thus could be used for _route initialization_ procedures.

    For example one could define route specific generators here:

        # place this in hook.pm file:
        # notices that we could tell GET from POST http methods here
        # using predefined $method variable

        sub list1 { 

            my $list;

            if ($method eq 'GET') {
                $list = | %w{ GET_foo GET_bar GET_baz } | 
            }elsif($method eq 'POST'){
                $list = | %w{ POST_foo POST_bar POST_baz } | 
            }else{
                die "method $method is not supported"
            }
            $list;
        }


        # now we could use it in swat data file
        generator:  list() 

- **route based perl cleanup hooks**

    Files located at `$route_directory/cleanup.pm`.

    This hooks is similar to route based startup hooks but \`required\` _in the end_ of a swat test.

## Bash hooks

Similar to perl hooks bash hooks are just a bash files \`sourced\` _before compilation_ of a swat test. 

There are 4 types of bash hooks:

- **project based bash hook**

    File located at `$project_root_directory/hook.bash`. 

    Project based bash hooks are applied for every route in project and could be used for _project initialization_ procedures.

- **route based bash hooks**

    Files located at `$project_root_directory/$route_directory/hook.bash`. 

    Routes based bash hooks are route specific hooks and could be used for _route initialization_ procedures.

- **global startup bash hook**

    File located at `$project_root_directory/startup.bash`. 

    Startup hook is executed before swat tests gets compiled, at the very beginning, at could be used for _global initialization_ procedures.

- **global cleanup bash hook**

    File located at `$project_root_directory/cleanup.bash`. 

    Cleanup hook is executed _after swat tests are executed_, at the very end, and could be used for _global cleanup_ procedures.

It is important to note that bash hooks are executed _after swat settings merge done_ , see  ["Swat Settings"](#swat-settings) section to get more
about swat settings.

## Predefined variables 

List of variables one may rely upon when writing perl/bash hooks:

- **http\_url**
- **curl\_params**
- **http\_meth** - `GET|POST|HEAD`
- **route\_dir**
- **project**

# Swat Compile and Runtime 

    - Execute *global startup bash hook*
    - Start of swat compilation phase
    - For every route gets compiled:
       -- Merge swat settings
        -- Set predefined variables
        -- Execute *project based bash hook*
        -- Execute *route based bash hook*
        -- Compile route test
    - The end of swat compilation phase
    - Start of swat execution phase. 
    - For every route test gets executed:
        -- Execute *project based perl startup hook*
        -- Execute *route based perl startup hook*
        -- Execute route test
        -- Execute *route based perl cleanup hook*
        -- Execute *project based perl cleanup hook*
    - The end of swat compilation phase
    - Execute *global cleanup bash hook*

# TAP

Swat produces output in [TAP](https://testanything.org/) format , that means you may use your favorite tap parsers to bring result to
another test / reporting systems, follow TAP documentation to get more on this. Here is example for converting swat tests into JUNIT format

    swat <project_root> <host> --formatter TAP::Formatter::JUnit

See also ["Prove settings"](#prove-settings) section.

# Command line tool

Swat is shipped as cpan package, once it's installed ( see ["Install"](#install) section ) you have a command line tool called **swat**, this is usage info on it:

    swat <project_root_dir|swat_package> <host:port> <prove settings>

- **host** - is base url for web application you run tests against, you also have to define swat routes, see DSL section.
- **project\_dir** - is a project root directory
- **swat\_package** - the name of swat package, see ["Swat Packages"](#swat-packages) section

## Default Host

Sometimes it is helpful to not setup host as command line parameter but define it at $project\_root/host file. For example:

    # let's create a default host for foo/bar project

    $ cat foo/bar/host
    foo.bar.com

    $ swat foo/bar/ # will run tests for foo.bar.com

## Debugging

set `swat_debug` environment variable to 1

## Running a subset of tests

It is possible to run a subset of swat test setting a `test_file` variable:

`test_file`={unix file path} . Test\_file path might be relative or absolute unix file path, internally swat just try to find all proper files using a trivial `unix find` 
construction:

    find $test_file -name  -type f -name get.txt -o -name post.txt -o -name put.txt

## Prove settings

Swat utilize [prove utility](http://search.cpan.org/perldoc?prove) to run tests, so all the swat options _are passed as is to prove utility_.
Follow [prove](http://search.cpan.org/perldoc?prove) utility documentation for variety of values you may set here.
Default value for prove options is  `-v`. Here is another examples:

- `-q -s` -  run tests in random and quite mode

# Swat Packages

Swat packages is portable archives of swat tests. It's easy to create your own swat packages and share with other. 

This is mini how-to on creating swat packages:

## Create swat package

Swat packages are _just cpan modules_. So all you need is to create cpan module distribution archive and upload it to CPAN.

The only requirement for installer is that swat data files should be installed into _cpan module directory_ at the end of install process. 
[File::ShareDir::Install](http://search.cpan.org/perldoc?File%3A%3AShareDir%3A%3AInstall) allows you to install 
read-only data files from a distribution and considered as best practice for such a things.

Here is example of Makefile.PL for [swat::mongodb package](https://github.com/melezhik/swat-packages/tree/master/mongodb-http):

    use inc::Module::Install;

    # Define metadata
    name           'swat-mongodb';
    all_from       'lib/swat/mongodb.pm';

    # Specific dependencies
    requires       'swat'         => '0.1.28';
    test_requires  'Test::More'   => '0';

    install_share  'module' => 'swat::mongodb', 'share';    

    license 'perl';

    WriteAll;

Here we create a swat package swat::mongodb with swat data files kept in the project\_root directory ./share and get installed into
`auto/share/module/swat-mongodb` directory.

Once we uploaded a module to CPAN repository we can use it: 

    $ cpan install swat::mongodb
    $ swat swat::mongodb 127.0.0.1:28017

Check out existed swat packages here - https://github.com/melezhik/swat-packages/

# Examples

./examples directory contains examples of swat tests for different cases. Follow README.md files for details.

# AUTHOR

[Aleksei Melezhik](mailto:melezhik@gmail.com)

# Home Page

https://github.com/melezhik/swat

# Thanks

To the authors of ( see list ) without who swat would not appear to light

- perl
- curl
- TAP
- Test::More
- prove

# COPYRIGHT

Copyright 2015 Alexey Melezhik.

This program is free software; you can redistribute it and/or modify it under the same terms as Perl itself.
