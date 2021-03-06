# sanitize-html

<a href="https://apostrophecms.org/"><img src="https://raw.github.com/punkave/sanitize-html-stream/master/logos/logo-box-madefor.png" align="right" /></a>


> NOTE: This is a fork of the excellent `sanitize-html` NPM to support streaming.  It is barely modified from `sanitize-html`.  For more details and the 
> original authors, see the bottom of this file.

`sanitize-html-stream` provides a simple HTML sanitizer with a clear API and the ability to process both streams and strings.  To use the streaming
approach, you can substitute a nodeJS Reader.  In that case, the library will return a Passthrough stream.  This is intended to be suitable for uploading
possibly large HTML documents to cloud storage without reading them into memory.  Unfortunately, I was unable to make the "exclusiveFilter" option work with streams (since it requires backtracking to the previous location in the result).  Therefore, it has been disabled.

`sanitize-html-stream` is tolerant. It is well suited for cleaning up HTML fragments such as those created by ckeditor and other rich text editors. It is especially handy for removing unwanted CSS when copying and pasting from Word.

`sanitize-html-stream` allows you to specify the tags you want to permit, and the permitted attributes for each of those tags.

If a tag is not permitted, the contents of the tag are still kept, except for `script`, `style` and `textarea` tags.

The syntax of poorly closed `p` and `img` elements is cleaned up.

`href` attributes are validated to ensure they only contain `http`, `https`, `ftp` and `mailto` URLs. Relative URLs are also allowed. Ditto for `src` attributes.

Allowing particular urls as a `src` to an iframe tag by filtering hostnames is also supported.

HTML comments are not preserved.

## Requirements

`sanitize-html-stream` is intended for use with Node. That's pretty much it. All of its npm dependencies are pure JavaScript. `sanitize-html-stream` is built on the excellent `htmlparser2` module.

## How to use

### Browser

Why would you want a node streams based library to run in the browser?  

### Node (Required)

Install module from console:

```bash
npm install sanitize-html-stream
```

Use it in your node app:

```js
var sanitizeHtml = require('sanitize-html-stream');

var dirty = 'some really tacky HTML';
var clean = sanitizeHtml(dirty);
```

That will allow our default list of allowed tags and attributes through. It's a nice set, but probably not quite what you want. So:

```js
// Allow only a super restricted set of tags and attributes
clean = sanitizeHtml(dirty, {
  allowedTags: [ 'b', 'i', 'em', 'strong', 'a' ],
  allowedAttributes: {
    'a': [ 'href' ]
  },
  allowedIframeHostnames: ['www.youtube.com']
});
```

Boom!

#### "I like your set but I want to add one more tag. Is there a convenient way?" Sure:

```js
clean = sanitizeHtml(dirty, {
  allowedTags: sanitizeHtml.defaults.allowedTags.concat([ 'img' ])
});
```

If you do not specify `allowedTags` or `allowedAttributes` our default list is applied. So if you really want an empty list, specify one.

#### "What are the default options?"

```js
allowedTags: [ 'h3', 'h4', 'h5', 'h6', 'blockquote', 'p', 'a', 'ul', 'ol',
  'nl', 'li', 'b', 'i', 'strong', 'em', 'strike', 'code', 'hr', 'br', 'div',
  'table', 'thead', 'caption', 'tbody', 'tr', 'th', 'td', 'pre', 'iframe' ],
allowedAttributes: {
  a: [ 'href', 'name', 'target' ],
  // We don't currently allow img itself by default, but this
  // would make sense if we did. You could add srcset here,
  // and if you do the URL is checked for safety
  img: [ 'src' ]
},
// Lots of these won't come up by default because we don't allow them
selfClosing: [ 'img', 'br', 'hr', 'area', 'base', 'basefont', 'input', 'link', 'meta' ],
// URL schemes we permit
allowedSchemes: [ 'http', 'https', 'ftp', 'mailto' ],
allowedSchemesByTag: {},
allowedSchemesAppliedToAttributes: [ 'href', 'src', 'cite' ],
allowProtocolRelative: true
```

#### "What if I want to allow all tags or all attributes?"

Simple! instead of leaving `allowedTags` or `allowedAttributes` out of the options, set either
one or both to `false`:

```js
allowedTags: false,
allowedAttributes: false
```

#### "What if I don't want to allow *any* tags?"

Also simple!  Set `allowedTags` to `[]` and `allowedAttributes` to `{}`.

```js
allowedTags: [],
allowedAttributes: {}
```

### "What if I want to allow only specific values on some attributes?"

When configuring the attribute in `allowedAttributes` simply use an object with attribute `name` and an allowed `values` array. In the following example `sandbox="allow-forms allow-modals allow-orientation-lock allow-pointer-lock allow-popups allow-popups-to-escape-sandbox allow-scripts"` would become `sandbox="allow-popups allow-scripts"`:

```js
        allowedAttributes: {
          iframe: [
            {
              name: 'sandbox',
              multiple: true,
              values: ['allow-popups', 'allow-same-origin', 'allow-scripts']
            }
          ]
```

With `multiple: true`, several allowed values may appear in the same attribute, separated by spaces. Otherwise the attribute must exactly match one and only one of the allowed values.

### Wildcards for attributes

You can use the `*` wildcard to allow all attributes with a certain prefix:

```javascript
allowedAttributes: {
  a: [ 'href', 'data-*' ]
}
```

Also you can use the `*` as name for a tag, to allow listed attributes to be valid for any tag:

```javascript
allowedAttributes: {
  '*': [ 'href', 'align', 'alt', 'center', 'bgcolor' ]
}
```

### htmlparser2 Options

`santizeHtml` is built on `htmlparser2`. By default the only option passed down is `decodeEntities: true` You can set the options to pass by using the parser option.

```javascript
clean = sanitizeHtml(dirty, {
  allowedTags: ['a'],
  parser: {
    lowerCaseTags: true
  }
});
```
See the [htmlparser2 wiki] (https://github.com/fb55/htmlparser2/wiki/Parser-options) for the full list of possible options.

### Transformations

What if you want to add or change an attribute? What if you want to transform one tag to another? No problem, it's simple!

The easiest way (will change all `ol` tags to `ul` tags):

```js
clean = sanitizeHtml(dirty, {
  transformTags: {
    'ol': 'ul',
  }
});
```

The most advanced usage:

```js
clean = sanitizeHtml(dirty, {
  transformTags: {
    'ol': function(tagName, attribs) {
        // My own custom magic goes here

        return {
            tagName: 'ul',
            attribs: {
                class: 'foo'
            }
        };
    }
  }
});
```

You can specify the `*` wildcard instead of a tag name to transform all tags.

There is also a helper method which should be enough for simple cases in which you want to change the tag and/or add some attributes:

```js
clean = sanitizeHtml(dirty, {
  transformTags: {
    'ol': sanitizeHtml.simpleTransform('ul', {class: 'foo'}),
  }
});
```

The `simpleTransform` helper method has 3 parameters:

```js
simpleTransform(newTag, newAttributes, shouldMerge)
```

The last parameter (`shouldMerge`) is set to `true` by default. When `true`, `simpleTransform` will merge the current attributes with the new ones (`newAttributes`). When `false`, all existing attributes are discarded.

You can also add or modify the text contents of a tag:

```js
clean = sanitizeHtml(dirty, {
  transformTags: {
    'a': function(tagName, attribs) {
        return {
            tagName: 'a',
            text: 'Some text'
        };
    }
  }
});
```
For example, you could transform a link element with missing anchor text:
```js
<a href="http://somelink.com"></a>
```
To a link with anchor text:
```js
<a href="http://somelink.com">Some text</a>
```

### Filters

Filters have been removed from the streaming version of this library as I haven't yet figured out a way to implement them without defeating the purpose of streaming.  (I haven't tried very hard as I don't need them at this moment.)

### Allowed CSS Styles

If you wish to allow specific CSS _styles_ on a particular element, you can do that with the `allowedStyles` option. Simply declare your desired attributes as regular expression options within an array for the given attribute. Specific elements will inherit whitelisted attributes from the global (\*) attribute. Any other CSS classes are discarded.

**You must also use `allowedAttributes`** to activate the `style` attribute for the relevant elements. Otherwise this feature will never come into play.

**When constructing regular expressions, don't forget `^` and `$`.** It's not enough to say "the string should contain this." It must also say "and only this."

**URLs in inline styles are NOT filtered by any mechanism other than your regular expression.**

```javascript
clean = sanitizeHtml(dirty, {
        allowedTags: ['p'],
        allowedAttributes: {
          'p': ["style"],
        },
        allowedStyles: {
          '*': {
            // Match HEX and RGB
            'color': [/^#(0x)?[0-9a-f]+$/i, /^rgb\(\s*(\d{1,3})\s*,\s*(\d{1,3})\s*,\s*(\d{1,3})\s*\)$/],
            'text-align': [/^left$/, /^right$/, /^center$/],
            // Match any number with px, em, or %
            'font-size': [/^\d+(?:px|em|%)$/]
          },
          'p': {
            'font-size': [/^\d+rem$/]
          }
        }
      });
```

### Allowed URL schemes

By default we allow the following URL schemes in cases where `href`, `src`, etc. are allowed:

```js
[ 'http', 'https', 'ftp', 'mailto' ]
```

You can override this if you want to:

```javascript
sanitizeHtml(
  // teeny-tiny valid transparent GIF in a data URL
  '<img src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==" />',
  {
    allowedTags: [ 'img', 'p' ],
    allowedSchemes: [ 'data', 'http' ]
  }
);
```

You can also allow a scheme for a particular tag only:

```javascript
allowedSchemes: [ 'http', 'https' ],
allowedSchemesByTag: {
  img: [ 'data' ]
}
```

And you can forbid the use of protocol-relative URLs (starting with `//`) to access another site using the current protocol, which is allowed by default:

```javascript
allowProtocolRelative: false
```

### Discarding the entire contents of a disallowed tag

Normally, with a few exceptions, if a tag is not allowed, all of the text within it is preserved, and so are any allowed tags within it.

The exceptions are:

`style`, `script`, `textarea`

If you wish to expand this list, for instance to discard whatever is found inside a `noscript` tag, use the `nonTextTags` option:

```javascript
nonTextTags: [ 'style', 'script', 'textarea', 'noscript' ]
```

Note that if you use this option you are responsible for stating the entire list. This gives you the power to retain the content of `textarea`, if you want to.

The content still gets escaped properly, with the exception of the `script` and `style` tags. *Allowing either `script` or `style` leaves you open to XSS attacks. Don't do that* unless you have good reason to trust their origin.

## About sanitize-html-stream, P'unk Avenue and Apostrophe

`sanitize-html-stream` is a streaming fork done in an afternoon of `sanitize-html`, which was created at [P'unk Avenue](http://punkave.com) for use in ApostropheCMS, an open-source content management system built on node.js. If you like `sanitize-html-stream` you should definitely [check out apostrophecms.org](http://apostrophecms.org).  I've posted this to npm and github because it seems there's no other **streaming** html sanitizer available and I should probably give back!  


