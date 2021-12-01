# nginxwaf format
nginxwaf uses a YAML-based declarative format to describe requests that should be allowed to reach your application. Everything described in this format is later turned into a set of nginx locations and rewrite rules.

## Basics
nginxwaf's format is a regular object.
Its most important key is **uri**. This key should point to an array of objects. Each of those objects associates a **pattern** with a **policy**.
Example:
```yaml
uri:
- pattern: /foo
  policy: foo_policy
- pattern: /bar
  policy: bar_policy
```
The pattern must be a pattern as described in the Patterns section whereas the policy can be either a policy object as described in the Policies section, or a string referring to such an object in the common.policy section.

## Patterns
Patterns describe normalised HTTP request URIs, as provided by nginx through its $uri variable.

Reminder about nginx's $uri: this variable is the HTTP request URI with:

 - query string removed;
 - duplicate slash characters merged;
 - path components canonicalised;
 - percent-encoded characters decoded.

By default, they are assumed to be plain, regular strings, but they will be treated as Perl-Compatible Regular Expressions (PCRE) if they feature regex metacharacters such as `\^$*+?()[]{}|`. Note: the presence of dot characters (`.`) is not enough for nginxwaf to consider a pattern as a regex.
Examples:
```yaml
uri:
- pattern: '/index.html'
  policy: static_file
- pattern: '/(?:addition|substraction|multiplication|division)'
  policy: arithmetic_operation
```
Note that it is not necessary to anchor pattern using `^` or `$`: this is handled by nginxwaf.

### Expansion
Additionally, writing patterns is made easier through an expansion mechanism.
Consider this example:
```yaml
# The "common" section is almost as important as the "uri" one since
# it enables declaring patterns and policies that can be reused multiple times:
common:
  pattern:
    positive_number: '[0-9]+'
uri:
- pattern: /addition/{positive_number}/{positive_number}
  policy: addition_policy
```
Pattern ids (here "positive_number") may feature alphanumeric characters, underscores, dashes and plus signs but must start with a letter so as to distinguish them from PCRE quantifiers (e.g. `a{5}` or `b{2,3}`). 

### Recursion
This mechanism works recursively, i.e. common patterns may refer to other common patterns:
```yaml
# The "common" section is almost as important as the "uri" one since
# it enables declaring patterns and policies that can be reused multiple times:
common:
  pattern:
    digit: '[0-9]'
    positive_number: '{digit}+'
uri:
- pattern: /addition/{positive_number}/{positive_number}
  policy: addition_policy
```
Limitation: recursion is limited to 100 levels (which ought to be enough for everyone ©).
### Lists
Common patterns may also be array of regular strings. This enables writing this:
```yaml
common:
  pattern:
    animal:
    - cat
    - dog
    - pig
    - bird
    - cow
    - sheep
    - rabbit
    - hare
```
instead of:
```yaml
common:
  pattern:
    animal: 'cat|dog|pig|bird|cow|sheep|rabbit|hare'
```
Note: values are escaped before being turned into a regex pattern.

### Extended/verbose/free-spacing patterns
After the expansion process, if a pattern happens to spread on multiple lines, it is automatically considered as an extended pattern (also known as verbose or free-spacing). Therefore, it is perfectly possible to write:
```yaml
uri:
- pattern: |-
    /{this}
    /{pattern}
    /{is}
    /{very}
    /{long}
    /{and}
    /{deserves}
    /{multiple}
    /{lines}
  policy: some_policy
```
### Order matters
nginxwaf turns each and every URI item into an nginx `location`. Specifically, it generates two kinds of locations:
* URI patterns that do not look like regexes are turned into exact match location, e.g. `location = /index.html`
* URI patterns that look like regexes are turned into regex locations, e.g. `location ~ "^(about|index|status)\.html$"`
nginx will reorder all exact match locations and try them first, so their order is absolutely irrelevant. Regex locations on the other hand should be carefully ordered as nginx will evaluate them sequentially until it finds a match.

## Policies
Policies are objects that describe what to check in the current HTTP request. All keys are optional. The following sections describe them individually but they can of course be combined.
### Methods
The `method` key must be an array of HTTP methods such as HEAD, GET, POST, PUT, DELETE, etc.
Example:
```yaml
uri:
- pattern: /foo
  policy:
    method:
    - GET
    - POST
- pattern: /baz
  policy:
    # It is possible to reference a common set of methods:
    method: all_but_webdav_and_connect
common:
  method:
    all_but_webdav_and_connect:
    - GET
    - HEAD
    - POST
    - PUT
    - PATCH
    - DELETE
```
If the current request's method does not match this list, HTTP 405 will be returned. This status code cannot be customised.
### Query string arguments
The `arg` key must be an array of objects. These objects may feature the following keys:
* `name` (mandatory): query string argument name;
* `pattern` (mandatory):  pattern to validate the query string argument value;
* `mandatory` (optional; defaults to False): whether or not the query string argument is mandatory;
* `status` (optional; default to global option `status`): HTTP status code to return if an invalid value is detected.

Example:
```yaml
uri:
- pattern: /
  policy:
    # An empty array means "no query string argument expected":
    arg: []
# /draw?animal=cow&count=4
- pattern: /draw
  policy:
    arg:
    # Direct description:
    - name: 'count'
      pattern: '{positive_integer}'
      mandatory: False
      status: 400
    # It is possible to reference a common argument:
    - animal
# /animate?animal=cat
- pattern: /animate
  policy:
    # It is possible to reference a common set of arguments:
    arg: animalonly
# /feed?animal=dog
- pattern: /feed
  policy:
    arg: animalonly
common:
  pattern:
    animal: 'cat|dog|pig|bird|cow|sheep|rabbit|hare'
    positive_integer: '[1-9][0-9]{0,3}'
  arg:
    animal:
      name: 'animal'
      pattern: '{animal}'
      mandatory: True
      status: 400
  argset:
    animal_only:
    - animal
```
Implementation note: query string argument values are reached through nginx's `$arg_` dynamic variable; nginx does not percent-decode the resulting values. Consequently, it is not possible to properly check UTF-8 values.
### Headers
The `header` argument must be an array of objects. These objects may feature the following keys:
* `name` (mandatory): header name;
* `pattern` (mandatory): pattern to validate the header value;
* `mandatory` (optional; defaults to False): whether or not the header is mandatory;
* `status` (optional; default to global option `status`): HTTP status code to return if an invalid value is detected.

Example:
```yaml
- pattern: /
  policy:
    header:
    - name: X-Event-UUID
      pattern: '{uuidv4}'
      mandatory: True
      status: 412
    # It is possible to reference a common header:
    - accept_html
- pattern: /browsers.html
  policy:
    # It is possible to reference a common set of arguments:
    header: accept_html_and_user_agent
common:
  pattern:
    hex: '[0-9A-Fa-f]'
    uuidv4: '{hex}{8}(?:-{hex}{4}){3}-{hex}{12}'
    year: '[12][0-9]{3}'
    month: '0[1-9]|1[012]'
    day: '[012][0-9]|3[01]'
    date: '{year}-{month}-{day}'
  header:
    accept_html:
      name: Accept
      pattern: '\btext/html\b'
      mandatory: True
      status: 406
  headerset:
    accept_html_and_user_agent:
    - name: Accept
      pattern: '.+'
      mandatory: True
    - name: User-Agent
      pattern: '.+'
      mandatory: True
```
Implementation note: query string argument values are reached through nginx's `$http_` dynamic variable.

### Cookies
The `cookie` argument must be an array of objects. These objects may feature the following keys:
* `name` (mandatory): cookie name;
* `pattern` (mandatory): pattern to validate the cookie value;
* `mandatory` (optional; defaults to False): whether or not the cookie is mandatory;
* `status` (optional; default to global option `status`): HTTP status code to return if an invalid value is detected.
Example:
```yaml
common:
  cookie:
    jsessionid:
      name: JSESSIONID
      pattern: '[0-9A-F]{32}'
      mandatory: True
    remember_me:
      name: remember_me
      pattern: 1
  cookieset:
    expected_cookies:
     # It is possible to reference a common cookie:
    - jsessionid
    - remember_me
uri:
- pattern: /special
  policy:
    cookie:
    - name: special_cookie
      pattern: '(?i)special_value'
      mandatory: True
      status: 412
- pattern: /user
  policy:
    # It is possible to reference a common set of cookies:
    cookie: expected_cookies
```
## Reusing objects
### Reusing policies
Like most objects described so far, policies can also be reused:
```yaml
common:
  policy:
    nocheck: {}
    public:
      method:
      - GET
      - HEAD
      arg: public_args
      cookie: public_cookies
      header: public_headers
    private:
      method:
      - GET
      - HEAD
      - POST
      arg: private_args
      cookie: private_cookies
      header: private_headers
uri:
- pattern: '/static/.+'
  policy: nocheck
- pattern: '/user/.+'
  policy: private
- pattern: '/.+'
  policy: public
```
### Summary
As shown in examples, most objects and set of objects can be declared once in the `common` section then reused almost anywhere else.
The table below summarizes available keys:
|Object type|parent|key     |Required?|common object key|common set key|
|-----------|------|--------|---------|-----------------|--------------|
|Patterns   |uri   |pattern |Mandatory|pattern          |-             |
|Policies   |uri   |policy  |Mandatory|policy           |-             |
|Methods    |policy|method  |Optional |method           |-             |
|Arguments  |policy|arg     |Optional |arg              |argset        |
|Headers    |policy|header  |Optional |header           |headerset     |
|Cookies    |policy|cookie  |Optional |cookie           |cookieset     |


## Options
Extra options provide better control on the nginx directives generated by nginxwaf:
```yaml
# If all application URIs start with the same prefix, then this prefix can
# be set through uri_prefix instead of being repeated in each and every URI object:
uri_prefix: my_app

# If WAF checks fail, the best reaction is to refuse further processing the request
# and generate an arbitrary response instead, typically a 4xx one. This setting
# defines which HTTP status code should be used.
# This value can be overridden for each specific checks (query string arguments,
# headers, cookies) except methods.
# Default value: 405
status: 444 # Special: instruct nginx to close the underlying TCP connection

#  If true, add an X-WAF-Debug HTTP header to responses that reflects the URI
# object / location chosen by nginx. It is useful during tests but should not
# be kept on production.
# Default value: False
debug: True

# If true, the very first generated nginx directive is:
#   uninitialized_variable_warn off;
# ... which prevents nginx from logging warnings whenever an uninitialized
# variable is tested.
# If false, this directive is not generated at all.
# Default value: True
uninitialized_variable_warn: False

# Name of the variable used to determine whether the current request was checked
# Default value: waf
variable: my_custom_filter

# Upon reception of the current request, its URI is prefixed with this value so
# as to jump to the WAF checks. After the request successfully passed these checks,
# the prefix is removed so as to jump back to nginx's regular processing. The
# chosen value should not conflict with application URIs.
# Default value: /waf
prefix: /G30b6pJjcsI3rzYbuFew/waf
```

