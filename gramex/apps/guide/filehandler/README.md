title: Gramex renders files

[gramex.yaml](../gramex.yaml) uses the [FileHandler][filehandler]
to display files. This folder uses the following configuration:

    :::yaml
    url:
      markdown:
        pattern: /$YAMLURL/(.*)               # Any URL under the current gramex.yaml folder
        handler: FileHandler                  # uses this handler
        kwargs:
          path: $YAMLPATH                     # Serve files from this YAML file's directory
          default_filename: README.md         # using README.md as default
          index: true                         # List files if README.md is missing

Any file under the current folder is shown as is. If a directory has a
`README.md`, that is shown by default.

`$YAMLURL` is replaced by the current URL's path (in this case, `/filehandler/`)
and `$YAMLPATH` is replaced by the directory of `gramex.yaml`.

**Note**: Gramex comes with a `default` URL handler that automatically serves
files from the home directory of your folder. To prevent that, override the
`default` pattern:

    :::yaml
    url:
      default:                          # This overrides the default URL handler
        pattern: ...


## Directory listing

`index: true` lists all files in the directory if the `default_filename` is
missing. To customise the directory listing, specify `index_template: filename`.
This file will be shown as HTML, with `$path` replaced by the directory's
absolute path, and `$body` replaced by a list of all files in that directory.

For example,

    :::yaml
      static:
        pattern: /static/(.*)                 # Any URL starting with /static/
        handler: FileHandler                  # uses this handler
        kwargs:
          path: static/                       # Serve files from static/
          default_filename: index.html        # using index.html as default
          index: true                         # List files if index.html is missing
          index_template: template.html       # Use template.html to list directory

Here is a trivial `template.html`. This must be placed in the same :

    :::html
    <h1>$path</h1>
    $body


## Redirecting files

You can specify any URL for any file. For example, to map the file
`filehandler/data.csv` to the URL `/filehandler/data`, use this configuration:

    :::yaml
    pattern: /filehandler/data    # The URL /filehandler/data
    handler: FileHandler          # uses this handler
    kwargs:
        path: filehandler/data.csv  # and maps to this file

You can also map regular expressions to file patterns. For example, to add a
`.yaml` extension automatically to a path, use:

    :::yaml
    url:
      yaml-extensions:
        pattern: "/yaml/(.*)"         # yaml/anything
        handler: FileHandler
        kwargs:
          path: "*.yaml"              # becomes anything.yaml, replacing the * here

For example, [yaml/gramex](yaml/gramex) actually renders [gramex.yaml](gramex.yaml).

To replace `.html` extension with `.yaml`, use:

    :::yaml
    url:
      replace-html-with-yaml:
        pattern: "/(.*)\\.html"       # Note the double backslash instead of single backslash
        handler: FileHandler
        kwargs:
          path: "*.yaml"              # The part in brackets replaces the * here


## Caching 

See how to cache [static files](../cache/#static-files)

## File patterns

If you want to map a subset of files to a folder, you can mark them in the
pattern. For example, this configuration maps `/style.css` and `/script.js` to
the home directory. To ensure that this takes priority over others, you can add
a higher value to the `priority` (which defaults to 0.)

    :::yaml
    url:
      assets:
        pattern: /(style.css|script.js)             # Any of these to URLs
        priority: 2                                 # Give it a higher priority
        handler: FileHandler                        # uses this handler
        kwargs:
          path: .                                   # Serve files from /

This can work across directories as well. For example, this maps the `static`
and `bower_components` and specifies a 1-day expiry for any files under them.

    :::yaml
    url:
      static-files:
        # Any file under the current directory, starting with bower_components/
        # or with static/ is mapped to a FileHandler
        pattern: /$YAMLURL/(bower_components/.*|static/.*)
        handler: FileHandler
        kwargs:
          path: $YAMLPATH/                          # Base is the current directory
          headers:
            Cache-Control: public, max-age=86400    # Cache publicly for 1 day


## Ignore files

To prevent certain files from ever being served, specify the
`handlers.FileHandler.ignore` setting. By default, this is:

    :::yaml
    handlers:
        FileHandler:
            ignore:
                - gramex.yaml           # Always ignore gramex.yaml in Filehandlers
                - ".*"                  # Hide dotfiles

The `gramex.yaml` file and all files beginning with `.` will be hidden. You can
customise this further via the `allow:` and `ignore:` configurations in the
handler. For example:

    :::yaml
    url:
        unsafe-handler:
            pattern: "/(.*)"
            handler: FileHandler
            kwargs:
                path: .
                ignore:
                    - secret.txt        # Also ignore secret.txt
                allow:
                    - gramex.yaml       # But allow gramex.yaml to be shown

In the above configuration, `secret.txt` will not be accessible, but
`gramex.yaml` will be.


## MIME types

The URL will be served with the MIME type of the file. CSV files have a MIME
type `text/csv` and a `Content-Disposition` set to download the file. You
can override these headers:

    :::yaml
    pattern: /filehandler/data
    handler: FileHandler
    kwargs:
        path: filehandler/data.csv
        headers:
            Content-Type: text/plain      # Display as plain text
            Content-Disposition: none     # Do not download the file


## Transforming content

Rather than render files as-is, the following parameters transform the markdown
into HTML:

    :::yaml
    # ... contd ...
      transform:
        "*.md":                                 # Any file matching .md
          function: markdown.markdown           #   Convert .md to html
          args: =content                        #   Pass the content as positional arg
          kwargs:                               #   Pass these arguments to markdown.markdown
            output_format: html5                #     Output in HTML5
          headers:                              #   Use these HTTP headers:
            Content-Type: text/html             #     MIME type: text/html

Any `.md` file will be displayed as HTML -- including this file (which is [README.md](README.md.source).)

Any transformation is possible. For example, this configuration converts YAML
into HTML using the [BadgerFish](http://www.sklar.com/badgerfish/) convention.

    :::yaml
    # ... contd ...
        "*.yaml":                               # YAML files use BadgerFish
          function: badgerfish                  # transformed via gramex.transforms.badgerfish()
          args: =content
          headers:
            Content-Type: text/html             # and served as HTML

Using this, the following file [page.yaml](page.yaml) is rendered as HTML:

    :::yaml
    html:
      "@lang": en
      head:
        meta:
          - {"@charset": utf-8}
          - {"@name": viewport, "@content": "width=device-width, initial-scale=1.0"}
        title: Page title
        link: {"@rel": stylesheet, "@href": /style.css}
      body:
        h1: Page constructed using YAML
        p: This file was created as YAML and converted into HTML using the BadgerFish convention.

Transforms take the following keys:

- **function**: The function to call as `function(*args, **kwargs)` using the
  `args` and `kwargs` below. You can use `=content` for the content and
  `=handler` for the handler in both `args` and `kwargs`.
- **args**: Positional parameters to pass. Defaults to the file contents.
- **kwargs**: Keyword parameters to pass.
- **encoding**: If blank, the file is treated as binary. The transform
  `function` MUST accept the content as binary. If you specify an encoding, the
  file is loaded with that encoding.
- **headers**: HTTP headers for the response.

Any function can be used as a transform. Gramex provides the following (commonly
used) transforms:

1. **template**. Use `function: template` to render the file as a [Tornado
   template][template]. Any `kwargs` passed will be sent as variables to the
   template. For example:

        :::yaml
        transform:
            "template.*.html":
                function: template            # Convert as a Tornado template
                args: =content                # Using the contents of the file (default)
                kwargs:                       # Pass it the following parameters
                    title: Hello world        # The title variable is "Hello world"
                    hander: =handler          # The handler variable is the RequestHandler

2. **badgerfish**. Use `function: badgerfish` to convert YAML files into HTML.
   For example, this YAML file is converted into a HTML as you would logically
   expect:

        :::yaml
        html:
          head:
            title: Sample file
          body:
            h1: Sample file
            p:
              - First paragraph
              - Second paragraph

## Templates

The `template` transform renders files as [Tornado templates][template]. To
serve a file as a Tornado template, use the following configuration:

    :::yaml
    url:
        template:
            pattern: /page                  # The URL /page
            handler: FileHandler            # displays a file
            kwargs:
                path: page.html             # named page.html
                transform:
                  "*.html":                 # Apply the transform to all HTML files
                    function: template      # Render page.html as a template
                    # The rest of this configuration is optional
                    kwargs:                 # You can pass additional variables to the template
                        title: "Variables"  # title is an explicit string
                        path: $YAMLPATH     # path is the current YAML file path
                        home: $HOME         # home is the YAML variable HOME (blank if not defined)
                        series: [a, b, c]   # series is a list of values

By default, the template receives a single keyword argument `handler`. You can
add other keyword arguments in `kwargs:` as shown above.

The file can contain any template feature. Here's a sample `page.html`.

    :::html
    <h1>{{ title }}</h1>
    <p>argument x is {{ handler.get_argument('x', None) }}</p>
    <p>path is: {{ path }}.</p>
    <p>home is: {{ home }}.</p>
    <ul>
        {% for item in series %}<li>{{ item }}</li>{% end %}
    </ul>

## Forms

If you're submitting forms using the POST method, you need to submit an
[_xsrf][xsrf] field that has the value of the `_xsrf` cookie. You can either
include it in the template using handlers' built-in `xsrf_token` property:

    :::html
    <form method="POST">
      <input type="hidden" name="_xsrf" value="{{ handler.xsrf_token }}">
    </form>

... or extract it dynamically using JavaScript. Here is an example that uses
[cookie.js](https://github.com/florian/cookie.js) and 
[jQuery](https://jquery.com/). Install them:

    :::shell
    bower install cookie jquery --save

Then extract the cookie and add a hidden input to your form:

    :::html
    <script src="bower_components/cookie/cookie.min.js"></script>
    <script src="bower_components/jquery/dist/jquery.min.js"></script>
    <script>
    $('<input>').attr({
      'type': 'hidden',
      'name': '_xsrf',
      'value': cookie.get('_xsrf')
    })
    .appendTo('form[method="POST"]')
    </script>

[xsrf]: http://www.tornadoweb.org/en/stable/guide/security.html#cross-site-request-forgery-protection

The XSRF cookie is automatically set when a FileHandler [template](#templates) accesses `handler.xsrf_token`. To set it explicitly, add `set_xsrf: true` to the handler kwargs like this:

    :::yaml
    url:
      name:
        pattern: ...              # When this page is visited,
        handler: ...              # no matter what the handler is,
        kwargs:
          ...
          set_xsrf: true          # set the xsrf cookie

You can disable XSRF in `gramex.yaml` (but this is **not recommended**):

    :::yaml
    app:
      settings:
        xsrf_cookies: false

For debugging without XSRF, start Gramex with a `--settings.xsrf_cookies=false` from the command line.

## File concatenation

You can concatenate multiple files and serve them as a single file. For example:

    :::yaml
    pattern: /contents
    handler: FileHandler
    kwargs:
        path:
            - heading.md
            - body.md
            - footer.md

This concatenates all files in `path` in sequence. If transforms are
specified, the transforms are applied before concatenation


[filehandler]: https://learn.gramener.com/gramex/gramex.handlers.html#gramex.handlers.FileHandler
[template]: http://www.tornadoweb.org/en/stable/template.html