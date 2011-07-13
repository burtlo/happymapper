HappyMapper
===========

Happymapper allows you to parse XML data and convert it quickly and easily into ruby data structures.

This project is a grandchild (a fork of a fork) of the great work done first by [jnunemaker](https://github.com/jnunemaker/happymapper) and then by [dam5s](http://github.com/dam5s/happymapper/).


Installation
------------

*Build the gem yourself:*

    $ git clone https://github.com/burtlo/happymapper
    $ cd happymapper
    $ gem build nokogiri-happymapper.gemspec
    $ gem install --local happymapper-X.X.X.gem

*For you [Bundler's](http://gembundler.com/) out there, you can add it to your Gemfile and then `bundle install`*

    gem 'happymapper', :git => "git://github.com/burtlo/happymapper.git"


Differences
-----------

  * [dam5s](http://github.com/dam5s/happymapper/) fork added [Nokogiri](http://nokogiri.org/) support
  * Fix for namespaces in instances of class [composition](https://github.com/burtlo/happymapper/commit/fd1e898c70f7289d2d2618d629b56f2f6623785c).
  * Fix for instances of XML where a [namespace is defined but no elements with that namespace are found](https://github.com/burtlo/happymapper/commit/9614221a80ff3bda18ff859aa751dff29cf52fd3).   
 
  * When a class definition contains a `xml_value=` definition then the method `to_xml` is defined on the class and will return the entire contents of the xml.
  
HappyMapper has support for `xml_content`, which will store the xml data within an element to this value. For example:

    class Article
      include HappyMapper
  
      tag 'Article'
      namespace 'article'
  
      attr_accessor :xml_content
  
      element :title, String
      element :author, String
    end

When parsing some xml that looks like this:

    <resultSet xmlns:article='http://www.wetpaint.com/alfresco/article'>
    <article:Article>
      <article:title>Title</article:title>
      <article:author>Author of Article</article:author>
    </article:Article>
    </resultSet>

You would be able to access this content:

    article = Article.parse(ARTICLE_XML,:single => true)
    puts article.xml_content  # => <article:title>Title</article:title>
                                 <article:author>Author of Article</article:author>
 
However, if you tried to parse that xml content later with any XML parser it fails because it did not know the namespaces. So now if you define `attr_writer :xml_value`

    class Article
      include HappyMapper
  
      tag 'Article'
      namespace 'article'
  
      attr_writer :xml_value
  
      element :title, String
      element :author, String
    end

When you call `to_xml` it will return the entire xml with the namespaces that might have been included in the element or even the parent elements.

    article = Article.parse(ARTICLE_XML,:single => true)
    puts article.xml_content  # => <article:Article xmlns:article='http://www.wetpaint.com/alfresco/article'>
                                     <article:title>Title</article:title>
                                     <article:author>Author of Article</article:author>
                                   </article:Article>

All namespaces that are available to the xml, from all their parent elements, are added, if they do not exist already to the xml. 

Adding a `to_xml` method may not be ideal in cases where you have your own `to_xml` as this representation will become out-of-date if you make some changes and want to serialize this back to xml.

 
Examples
--------

## Element Mapping

Let's start with a simple example to get our feet wet. Here we have a simple example of XML that defines some address information:

    <address>
      <street>Milchstrasse</street>
      <housenumber>23</housenumber>
      <postcode>26131</postcode>
      <city>Oldenburg</city>
      <country code="de">Germany</country>
    </address>

Happymapper will let you easily model this information as a class:

    require 'happymapper'

    class Address
      include HappyMapper

      tag 'address'
      element :street, String, :tag => 'street'
      element :postcode, String, :tag => 'postcode'
      element :housenumber, Integer, :tag => 'housenumber'
      element :city, String, :tag => 'city'
      element :country, String, :tag => 'country'
    end

To make a class HappyMapper compatible you simply `include HappyMapper` within the class definition. This takes care of all the work of defining all the speciality methods and magic you need to get running. As you can see we immediately start using these methods.

* `tag` matches the name of the XML tag name 'address'.

* `element` defines accessor methods for the specified symbol (e.g. `:street`,`:housenumber`) that will return the class type (e.g. `String`,`Integer`) of the XML tag specified (e.g. `:tag => 'street'`, `:tag => 'housenumber'`).
    
When you define an element with an accessor with the same name as the tag, this is the case for all the examples above, you can omit the `:tag`. These two element declaration are equivalent to each other:

    element :street, String, :tag => 'street'
    element :street, String

Including the additional tag element is not going to hurt anything and in some cases will make it absolutely clear how these elements map to the XML. However, once you know this rule, it is hard not to want to save yourself the keystrokes.

Instead of `element` you may also use `has_one`:

    element :street, String, :tag => 'street'
    element :street, String
    has_one :street, String

These three statements are equivalent to each other.

## Parsing

With the mapping of the address XML articulated in our Address class it is time to parse the data:

    address = Address.parse(ADDRESS_XML_DATA, :single => true)
    puts address.street 

Assuming that the constant `ADDRESS_XML_DATA` contains a string representation of the address XML data this is fairly straight-forward save for the `parse` method.

The `parse` method, like `tag` and `element` are all added when you included HappyMapper in the class. Parse is a wonderful, magical place that converts all these declarations that you have made into the data structure you are about to know and love.

But what about the `:single => true`? Right, that is because by default when your object is all done parsing it will be an array. In this case an array with one element, but an array none the less. So the following are equivalent to each other:

    address = Address.parse(ADDRESS_XML_DATA).first
    address = Address.parse(ADDRESS_XML_DATA, :single => true)

The first one returns an array and we return the first instance, the second will do that work for us inside of parse.

## Multiple Elements Mapping

What if our address XML was a little different, perhaps we allowed multiple streets:

    <address>
      <street>Milchstrasse</street>
      <street>Another Street</street>
      <housenumber>23</housenumber>
      <postcode>26131</postcode>
      <city>Oldenburg</city>
      <country code="de">Germany</country>
    </address>

Similar to `element` or `has_one`, the declaration for when you have multiple elements you simply use:

    has_many :streets, String, :tag => 'street'

Your resulting `street` method will now return an array.

    address = Address.parse(ADDRESS_XML_DATA, :single => true)
    puts address.streets.join('\n')
    
Imagine that you have to write `street.join('\n')` for the rest of eternity throughout your code. It would be a nightmare and one that you could avoid by creating your own convenience method.

     require 'happymapper'

     class Address
       include HappyMapper

       tag 'address'
       
       has_many :streets, String, :tag => 'street'
       
       def street
         streets.join('\n')
       end
       
       element :postcode, String, :tag => 'postcode'
       element :housenumber, String, :tag => 'housenumber'
       element :city, String, :tag => 'city'
       element :country, String, :tag => 'country'
     end

Now when we access `street` we get a single value, but we still have `streets` if we ever need to the values as an array.


## Attribute Mapping

    <address location='home'>
      <street>Milchstrasse</street>
      <street>Another Street</street>
      <housenumber>23</housenumber>
      <postcode>26131</postcode>
      <city>Oldenburg</city>
      <country code="de">Germany</country>
    </address>

Attributes are absolutely the same as `element` or `has_many`

    attribute :location, String, :tag => 'location
    
Again, you can omit the tag if the attribute accessor symbol matches the name of the attribute.


### Attributes On Empty Child Elements

    <feed xml:lang="en-US" xmlns="http://www.w3.org/2005/Atom">
      <id>tag:all-the-episodes.heroku.com,2005:/tv_shows</id>
      <link rel="alternate" type="text/html" href="http://all-the-episodes.heroku.com"/>
      <link rel="self" type="application/atom+xml" href="http://all-the-episodes.heroku.com/tv_shows.atom"/>
      <title>TV Shows</title>
      <updated>2011-07-10T06:52:27Z</updated>
    </feed>

In this case you would need to map an element to a new `Link` class just to access `<link>`s attributes, except that there is an alternate syntax. Instead of

    class Feed
    # ....
      has_many :links, Link, :tag => 'link', :xpath => '.'
    end

    class Link
      include HappyMapper

      attribute :rel, String
      attribute :type, String
      attribute :href, String
    end

You can drop the `Link` class and simply replace the `has_many` on `Feed` with

    element :link, String, :single => false, :attributes => { :rel => String, :type => String, :href => String }

As there is no content, the type given for `:link` (`String` above) is irrelevant, but `nil` won't work and other types may try to perform typecasting and fail. You can omit the :single => false for elements that only occur once within their parent.

This syntax is most appropriate for elements that (a) have attributes but no content and (b) only occur at only one level of the heirarchy. If `<feed>` contained another element that also contained a `<link>` (as atom feeds generally do) it would be DRY-er to use the first syntax, i.e. with a separate `Link` class.


## Class composition

Our address has a country and that country element has a code. Up until this point we neglected it as we declared a `country` as being a `String`.

    <address location='home'>
      <street>Milchstrasse</street>
      <street>Another Street</street>
      <housenumber>23</housenumber>
      <postcode>26131</postcode>
      <city>Oldenburg</city>
      <country code="de">Germany</country>
    </address>

Well if we only going to parse country, on it's own, we would likely create a class mapping for it.

    class Country
      include HappyMapper
  
      tag 'country'
  
      attribute :code, String
      text_node :name, String
    end

We are utilizing an `attribute` declaration and a new declaration called `text_node`.

* `text_node` is used when you want the text contained within the element

Awesome, now if we were to redeclare our `Address` class we would use our new `Country` class.

    class Address
      include HappyMapper

      tag 'address'
  
      has_many :streets, String, :tag => 'street'
  
      def street
        streets.join('\n')
      end
  
      element :postcode, String, :tag => 'postcode'
      element :housenumber, String, :tag => 'housenumber'
      element :city, String, :tag => 'city'
      element :country, Country, :tag => 'country'
    end
  
Instead of `String`, `Boolean`, or `Integer` we say that it is a `Country` and HappyMapper takes care of the details of continuing the XML mapping through the country element.

    address = Address.parse(ADDRESS_XML_DATA, :single => true)
    puts address.country.code
  
A quick note, in the above example we used the constant `Country`. We could have used `'Country'`. The nice part of using the latter declaration, enclosed in quotes, is that you do not have to define your class before this class. So Country and Address can live in separate files and as long as both constants are available when it comes time to parse you are golden.

## Custom XPATH

### Has One, Has Many

Getting to elements deep down within your XML can be a little more work if you did not have xpath support. Consider the following example:

    <media>
      <gallery>
        <title href="htttp://fishlovers.org/friends">Friends Who Like Fish</title>
        <picture>
          <name>Burtie Sanchez</name>  
          <img>burtie01.png</img>
        </picture>
      </gallery>
      <picture>
        <name>Unsorted Photo</name>  
        <img>bestfriends.png</img>
      </picture>
    </media>

You may want to map the sub-elements contained buried in the 'gallery' as top level items in the media. Traditionally you could use class composition to accomplish this task, however, using the xpath attribute you have the ability to shortcut some of that work.

    class Media
      include HappyMapper
  
      has_one :title, String, :xpath => 'gallery/title'
      has_one :link, String, :xpath => 'gallery/title/@href'
    end


### Subclasses
I ran into a case where I wanted to capture all the pictures that were directly under media, but not the ones contained within a gallery.

    <media>
      <gallery>
        <picture>
          <name>Burtie Sanchez</name>  
          <img>burtie01.png</img>
        </picture>
      </gallery>
      <picture>
        <name>Unsorted Photo</name>  
        <img>bestfriends.png</img>
      </picture>
    </media>
    
The following `Media` class is where I started:

    require 'happymapper'

    class Media
      include HappyMapper
  
      has_many :galleries, Gallery, :tag => 'gallery'
      has_many :pictures, Picture, :tag => 'picture'
    end

However when I parsed the media xml the number of pictures returned to me was 2, not 1.

    pictures = Media.parse(MEDIA_XML,:single => true).pictures
    pictures.length.should == 1   # => Failed Expectation

I was mistaken and that is because, by default the mappings are assigned XPATH './/' which is requiring all the elements no matter where they can be found. To override this you can specify an XPATH value for your defined elements.

    has_many :pictures, Picture, :tag => 'picture', :xpath => '/media'

`/media` states that we are only interested in pictures that can be found directly under the media element. So when we parse again we will have only our one element.


## Namespaces

Obviously your XML and these trivial examples are going to easy to map and parse because they lack the treacherous namespaces that befall most XML files.

Perhaps our `address` XML is really swarming with namespaces:

    <prefix:address location='home' xmlns:prefix="http://www.unicornland.com/prefix">
      <prefix:street>Milchstrasse</prefix:street>
      <prefix:street>Another Street</prefix:street>
      <prefix:housenumber>23</prefix:housenumber>
      <prefix:postcode>26131</prefix:postcode>
      <prefix:city>Oldenburg</prefix:city>
      <prefix:country code="de">Germany</prefix:country>
    </prefix:address>

Here again is our address example with a made up namespace called `prefix` that comes direct to use from unicornland, a very magical place indeed. Well we are going to have to do some work on our class definition and that simply adding this one liner to `Address`:

    namespace 'prefix'
    
Of course, if that is too easy for you, you can append a `:namespace => 'prefix` to every one of the elements that you defined. 

    has_many :streets, String, :tag => 'street', :namespace => 'prefix'
    element :postcode, String, :tag => 'postcode', :namespace => 'prefix'
    element :housenumber, String, :tag => 'housenumber', :namespace => 'prefix'
    element :city, String, :tag => 'city', :namespace => 'prefix'
    element :country, Country, :tag => 'country', :namespace => 'prefix'
    
I definitely recommend the former, as it saves you a whole hell of lot of typing. However, there are times when appending a namespace to an element declaration is important and that is when it has a different namespace then `namespsace 'prefix'`.

Imagine that our `country` actually belonged to a completely different namespace.

    <prefix:address location='home' xmlns:prefix="http://www.unicornland.com/prefix"
    xmlns:prefix="http://www.trollcountry.com/different">
      <prefix:street>Milchstrasse</prefix:street>
      <prefix:street>Another Street</prefix:street>
      <prefix:housenumber>23</prefix:housenumber>
      <prefix:postcode>26131</prefix:postcode>
      <prefix:city>Oldenburg</prefix:city>
      <different:country code="de">Germany</different:country>
    </prefix:address>

Well we would need to specify that namespace:

    element :country, Country, :tag => 'country', :namespace => 'different'
    
With that we should be able to parse as we once did.