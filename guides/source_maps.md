# Source Maps

This is mostly a guide for developers of Sprockets, contents may change without notice.

## What is a source map?

In production Sprockets combines files together and minifies them when possible. This makes serving HTTP 1.x traffic faster, but if there is an error in your assets, it becomes very difficult to debug. In Rails asset pipeline it was the convention to not concatenate these files in development, so instead of serving 1 file you might see 10 or so. A source map is a standard that allows assets bundled into one file to declare a "source map" that lets browsers know what code came from what sources.

- [Source Map 3 proposal](https://docs.google.com/document/d/1U1RGAehQwRypUTovF1KRlpiOFze0b-_2gc6fAH0KY0k/edit)
- [Mozilla source-map library](https://github.com/mozilla/source-map/)

So if a source map is used, the exact same method of concatenation and minifciation in production can be used in development. This encourage developers to use standardized tools that are adopted across browsers to debug their assets.

## Source Map Detection

When an asset is served to the browser it lets the browser know if a source map is available by adding a special comment to the bottom. For javascript files with a the comment starts with

```js
//# sourceMappingURL=
```

For example a javascript file like appli caiton.js was being served from `public/assets/application.js` then it might have a map file in `public/assets/application.js.map`. In that case the comment could either be a full path


```js
//# sourceMappingURL=/assets/application.js.map
```

Or it could be relative to the parent's directory:


```js
//# sourceMappingURL=application.js.map
```

Css files have a different comment specification

```
/*# sourceMappingURL=application.css.map */
```

When this comment is served, the browser can make an additional request to that location to get the source map associated with the file

## Encode/Decode Source Map

Mozilla maintains a node module that can encode and decode source maps. [Mozilla source-map library](https://github.com/mozilla/source-map/). We are considering this the source implementation against which sprockets can be compared.

First you'll need `npm` installed, google it.

First we will build a source map using the uglify-js library

```
$ npm install uglify-js
uglify-js@2.6.1 node_modules/uglify-js
├── uglify-to-browserify@1.0.2
├── async@0.2.10
├── source-map@0.5.3
└── yargs@3.10.0 (decamelize@1.1.1, window-size@0.1.0, camelcase@1.2.1, cliui@2.1.0)
```

Now we need an original javascript file:

```
$ cat foo.js
var foo = "foo";
var bar = "bar";
```

We can run uglifyier on this file to generate a smaller version as well as a source map file by specifying the file name with the `--source-map` flag.

```
$ uglifyjs foo.js --source-map foo.js.map
var foo="foo";var bar="bar";
//# sourceMappingURL=foo.js.map
```

Now you can view this file:

```
$ cat foo.js.map
{
  "version":  3,
  "sources":  ["foo.js"],
  "names":    ["foo","bar"],
  "mappings": "AAAA,GAAIA,KAAM,KACV,IAAIC,KAAM"
}
```

Next you'll need to install the `source-map` library


```sh
$ npm install source-map
source-map@0.5.3 node_modules/source-map
```

now we'll need simple node script that parses this file:


```js
var sourceMap = require('source-map');
var fs        = require('fs');

fs.readFile('./foo.js.map', 'utf8', function (err, data) {
  if (err) {
    return console.log(err);
  }
  var smc = new sourceMap.SourceMapConsumer(data);

  smc.eachMapping(function(m) {
    console.log(m);
  });
});
```

Save this in `read-source-map.js` when you run this file:

```
$ node read-source-map.js
{ source: 'foo.js',
  generatedLine: 1,
  generatedColumn: 0,
  originalLine: 1,
  originalColumn: 0,
  name: null }
{ source: 'foo.js',
  generatedLine: 1,
  generatedColumn: 3,
  originalLine: 1,
  originalColumn: 4,
  name: 'foo' }
{ source: 'foo.js',
  generatedLine: 1,
  generatedColumn: 8,
  originalLine: 1,
  originalColumn: 10,
  name: null }
{ source: 'foo.js',
  generatedLine: 1,
  generatedColumn: 13,
  originalLine: 2,
  originalColumn: 0,
  name: null }
{ source: 'foo.js',
  generatedLine: 1,
  generatedColumn: 17,
  originalLine: 2,
  originalColumn: 4,
  name: 'bar' }
{ source: 'foo.js',
  generatedLine: 1,
  generatedColumn: 22,
  originalLine: 2,
  originalColumn: 10,
  name: null }
```

Each of these correspond to an object in our javascript file. If we look at the `foo` variable:

```
{ source: 'foo.js',
  generatedLine: 1,
  generatedColumn: 3,
  originalLine: 1,
  originalColumn: 4,
  name: 'foo'
}
```


## Source Map file

If we generate a source map for a 1 line javascript file that is not concatenated (it is generated by only one file) we can get a sense of a simple source map. For example if we generate a source map of `foo.js` which has these contents:

```js
var foo;
```

Then the resultant `foo.js.map` will be

```js
{
  "version": 3,
  "sources": ["foo.js"],
  "names":   [ ],
  "mappings": "AAAA,GAAIA"
}
```

- `version` The version of the source map specification we are using. The current is version 3.
- `sources` An array of source files, these are the files used to generate `foo.js` if there were more files concatenated we would be expected to see multiple entries here.
- `names` Names of functions if available
- `mappings` The secret sauce, this includes a VLQ base 64 encoded string that tells the browser how to map lines and locations in the generated file to files, in our case `foo.js`

## Understanding Mappings


Mappings are encoded from the version 3 spec. They use a [Variable Length Quantity](https://en.wikipedia.org/wiki/Variable-length_quantity) of Base 64 encoded strings. This allows us to represent arbitrarily large strings. It works like this:


### VLQ Base 64 bit mappings

We can represent integers in base64. First we generate an array of valid base64 characters

```ruby
BASE64_DIGITS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'.split('')
```

Then we can generate a hash of those characters to their coresponding numeric value


```ruby
BASE64_VALUES = (0...64).each_with_object({}) { |i, hash| hash[BASE64_DIGITS[i]] = i }
```

So the value of "A" would be `0` and "9" would be `61`. So now we can represent numbers 0 up to 64 with only one character. Since we need to go higher than 64 digits, the VLQ lets us use a bit inside of the base64 bit value to determine if we continue or stop. In the spec it says that the base64 digit can contain 6 bits of data. The 6th bit is the "continuation" bit which tells us to either stop or keep going.

We can determine if a continuation bit is set with bit shifting and maskign. So if we have 6 bits repersenting 1: "000001" it would take us 5 shifts to represent "100000" which would be only the continuation bit.

```ruby
VLQ_BASE_SHIFT = 5
```

We then shift that position onto 1 to determine our mask

```ruby
VLQ_CONTINUATION_BIT = 1 << VLQ_BASE_SHIFT
```

From the Ruby 2.2.3 docs this will shift the fixnum on the left (1) by the count positions on the right (VLQ_BASE_SHIFT) which is 5. This generates the number 32. We can verify this is our 6th bit with a little inspection

```ruby
VLQ_CONTINUATION_BIT.to_s(2)
# => "100000"
```

This works because we `to_s` accepts a base, by passing in a base of 2 we are returning binary representation.

Now we can determine if an iteger has the continuation bit set by bit masking. Using a [bitwise &](http://ruby-doc.org/core-2.2.3/Fixnum.html#method-i-26) we mask out all the bits to zero except for the first one. If the result returned is 0 it means that the bit is not set and processing should not be continued:


```ruby
digit = BASE64_VALUES["A"]
digit & VLQ_CONTINUATION_BIT
# => 0
```

So now we know how to detect for continuation bits, but how do we actually use them? In the previous example our mapping returned "AAAA". Since an "A" maps to zero this would generate an array like


```
str = "A"
vlq_decode(str)
# => [0]

str = "AAAA"
vlq_decode(str)
# => [0, 0, 0, 0]
```

The first character that has its continuation bit set is lowercase "g". A lowercase "g" returns a value of `32`. The value of`vlq_decode("gA")` turns out to be equal to 0. So what would `vlq_decode("gB")` result in? To understand this we need to look a the whole method:

```ruby
def vlq_decode(str)
  result = []
  chars = str.split('')
  while chars.any?
    vlq = 0
    shift = 0
    continuation = true
    while continuation
      char = chars.shift
      raise ArgumentError unless char
      digit = BASE64_VALUES[char]
      continuation = false if (digit & VLQ_CONTINUATION_BIT) == 0
      digit &= VLQ_BASE_MASK
      vlq   += digit << shift
      shift += VLQ_BASE_SHIFT
    end
    result << (vlq & 1 == 1 ? -(vlq >> 1) : vlq >> 1)
  end
  result
end
```

For the first inner loop we would get a digit of `32` for the character "g". We see the continuation bit is set, so we keep `continuation variable to `true`. We then mask and set the `digit` with `VLQ_BASE_MASK` which is

```
VLQ_BASE_MASK = VLQ_BASE - 1
# => 31
31.to_s(2)
# => "111111"
```

So then

```ruby
digit &= VLQ_BASE_MASK
# => 0
digit.to_s(2)
# => "0"
# or "000000"
```

Whe then generate a `vl` by shifting the digit with the default value of `shift` which is 0

```
vlq   += digit << shift
# => 0
```

So the value for this iteration would be zero.

Finally we update the shift value:

```
shift += VLQ_BASE_SHIFT
# => 5
```

Since continuation is set to true, we go on to the next character "B".


```
digit = BASE64_VALUES["B"]
# => 1
continuation = false if (digit & VLQ_CONTINUATION_BIT) == 0
# => false
digit &= VLQ_BASE_MASK
# => 1
vlq   += digit << shift
# => 32
shift += VLQ_BASE_SHIFT
# => 10
```

Now we have no more characters and continuation is false. We then add to our result. We use the first bit to check for sign so "000001" is a negative number. Since that is not the case, we shift the value of the vlq to the right so 32 which is "100000" becomes "010000" which is:

```
vlq >> 1
# => [16]
```

This is our result:

```
vlq_decode("gC")
# => [16]
```

So what would `vlq_decode("gC")` generate? The first iteration will be the same.

```
digit = BASE64_VALUES["g"]
# => 32
continuation = false if (digit & VLQ_CONTINUATION_BIT) == 0
# continuation does not change, still true
digit &= VLQ_BASE_MASK
# => 0
vlq   += digit << shift
# => 0
shift += VLQ_BASE_SHIFT
# => 5
```

The second time the only thing htat is different with `"C"` is the digit and vlq:


```
digit &= VLQ_BASE_MASK
# => 2
vlq   += digit << shift
# => 64
```

The vlq `64` does not have it's first bit set so it is positive. We shift this right by 1 and since we lose a bit, we get `32`.


```
vlq_decode("gC")
# => [32]
```

So our initial output from `foo.js` is

```
vlq_decode("AAAA")
#=> [0, 0, 0, 0]
vlq_decode("GAAIA")
#=> [3, 0, 0, 4, 0]
```


## Sprockets Internal Map support

Internally sprockets stores maps as hashes that look like this:
We need to be able to generate information like

```
{
  :source=>"example.coffee",
  :generated=>[6, 2],
  :original=>[2, 0],
  :name=>"number"
}
```

This would be for the case where `example.coffee` has a value called `number`

```
# Assignment:
number   = 42
```

In the original document is on the 2nd line, and it's first character is on the 1st column, so it starts on the 0th column.

Compiling this file will generate a coffee script file that starts with this:

```
// Generated by CoffeeScript 1.8.0
(function() {
  var cubes, list, math, num, number, opposite, race, square,
    __slice = [].slice;

  number = 42;
  # ...
```

You can see that the generated `number` variable gets assigned on the 6th line and the first character is on the 3rd column so it starts on the 2nd column.


## Mapping format



The mapping field contains VLQ encoded strings as well as commas "," and semicolons ";" that are used as delimiters.



```
// A single base 64 digit can contain 6 bits of data. For the base 64 variable
// length quantities we use in the source map spec, the first bit is the sign,
// the next four bits are the actual value, and the 6th bit is the
// continuation bit. The continuation bit tells us whether there are more
// digits in this value following this digit.
//
//   Continuation
//   |    Sign
//   |    |
//   V    V
//   101011
```

I have no idea

The “mappings” data is broken down as follows:
- each group representing a line in the generated file is separated by a ”;”
- each segment is separated by a “,”
- each segment is made up of 1,4 or 5 variable length fields.

Confused? I was.



Lots of this is raw notes, take with a grain of salt.
