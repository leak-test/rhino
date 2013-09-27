# Rhino - a Ruby ORM for HBase

Rhino is a Ruby object-relational mapping (ORM) for HBase[http://www.hbase.org].

### Support & contact

Author: Quinn Slack qslack@qslack.com[mailto:qslack@qslack.com]

Contributors: Dru Jensen

## Getting started

### Download Rhino

  git clone git://github.com/sqs/rhino.git

### Installing HBase and Thrift

Since Rhino uses the HBase Thrift API, you must first install both HBase and Thrift. Downloading the latest trunk revisions of each is recommended, but if you encounter problems, try using the latest stable release instead. Here are the basic steps for installing both.

#### Installing HBase
  
  svn co http://svn.apache.org/repos/asf/hadoop/hbase/trunk hbase-core-trunk
  cd hbase-core-trunk
  ant
  
{More installation instructions}[http://wiki.apache.org/hadoop/Hbase/10Minutes] are available on the HBase Wiki.

#### Installing Thrift

Thrift[http://developers.facebook.com/thrift/] also requires the Boost C++ libraries; you'll have to get those on your own if your system does not have them.

  svn co http://svn.facebook.com/svnroot/thrift/trunk/ thrift-trunk
  cd thrift-trunk
  ./bootstrap.sh
  ./configure && make && sudo make install
  cd lib/rb
  sudo ruby setup.rb

#### Starting the HBase and Thrift servers

Once you have installed HBase and Thrift, start HBase, then start the Thrift server. From the root HBase directory, run these commands:

  bin/start-hbase.sh
  bin/hbase thrift start
  
Both servers need to be running to use Rhino. Occasionally, the Thrift server will be unable to connect to HBase. In that case, stop the Thrift server (ctrl-C), stop the HBase server (<tt>bin/stop-hbase.sh</tt>), and then rerun the above commands to restart both. To verify that HBase is running, try running the HBase shell (<tt>bin/hbase shell</tt>).

#### Loading Rhino

Since Rhino is not yet packaged as a gem, you will have to <tt>require 'PATH_TO_RHINO/lib/rhino.rb'</tt> in your scripts.
  
## Usage  

### Connect to HBase

The following code points Rhino to the Thrift server you just started (which by default listens on localhost:9090).
  
  Rhino::Model.connect('localhost', 9090)
  
### Describe your table

A class definition like:

  class Page < Rhino::Model
    include Rhino::Constraints

    column_family :title
    column_family :contents
    column_family :links
    column_family :meta
    column_family :images
    
    alias_attribute :author, 'meta:author'

    has_many :links, Link
    has_many :images, Image

    constraint(:title_required) { |page| page.title and !page.title.empty? }
  end

...is mapped to the following HBase table (described in {HBase Query Language}[http://wiki.apache.org/lucene-hadoop/HBase/HBaseShell]

  CREATE TABLE pages(title:, contents:, links:, meta:, images:);
  
Or, in version 0.2's JRuby shell language:
  
  create 'pages', 'title', 'contents', 'links', 'meta', 'images'

### Basic operations

#### Getting records

  page # Page.get('some-page')
  all_pages # Page.get_all()
  
#### Creating new records

  # data can be specified in the second argument of Page.new...
  page # Page.new('the-row-key', {:title#>"my title"})
  # ...or as attributes on the model
  page.contents # "<p>welcome</p>"
  page.save

#### Reading and updating attributes

  page # Page.get('some-key')
  puts "the old title is: #{page.title}"
  page.title # "another title"
  page.save
  puts "the new title is: #{page.title}"

You can also read from and write to specific columns in a column family.
Since we already defined the <tt>meta:</tt> column family, Rhino knows we want to set the <tt>meta:author</tt> column:

  page # Page.get('some-key')
  page.meta_author # "John Doe"
  page.save
  puts "the author is: #{page.meta_author}"
  
### has_many and belongs_to

In the model definition above, we stated that a Page <tt>has_many :links</tt> and <tt>has_many :images</tt>. We can 
define what a Link and an Image is with greater detail now.

  class Link < Rhino::Cell
    belongs_to :page
  
    def url
      url_parts # key.split('/')
      backwards_host # url_parts.shift
      path # url_parts.join('/')
      host # backwards_host.split('.').reverse.join('.')
      "http://#{host}/#{path}"
    end
  end

  class Image < Rhino::Cell
    belongs_to :page
  end
  
Now that we've defined <tt>Link</tt> and <tt>Image</tt>, we can work with them easily. The following code adds a link to the page <tt>com.example</tt>, which when written to HBase becomes a cell in the <tt>links:</tt> column family named <tt>links:com.google</tt> with the contents <tt>search engine</tt>.

  page # Page.get('com.example')
  page.links.create('com.google', 'search engine')
  
You can also iterate over the collection of links. In this example, we use <tt>Link#url</tt>, a method we defined on the Link class
to convert from the common <tt>com.example/path</tt> URL storage style to <tt>example.com/path</tt>.

  page.links.each do |link|
    puts "Link to #{link.url} with text: '#{link.contents}'"
  end
  
You can also get a specific link.

  google_link_text # page.get('com.google').contents

### Setting timestamps and retrieving by timestamp
  
First, let's create some Pages with different timestamps.

  a_week_ago # Time.now - 7 * 24 * 3600
  a_month_ago # Time.now - 30 * 24 * 3600
  
  newer_page # Page.create('google.com', {:title#>'newer google'}, {:timestamp#>a_week_ago})
  older_page # Page.create('google.com', {:title#>'older google'}, {:timestamp#>a_month_ago})
  
Now you can <tt>get</tt> by the timestamps you just set.

  Page.get('google.com', :timestamp#>a_week_ago).title # #> "newer google"
  Page.get('google.com', :timestamp#>a_month_ago).title # #> "older google"
  
If you call <tt>get</tt> with no arguments, you will get the most recent Page.

  Page.get('google.com').title # #> "newer google"
  
## More information

Read the specs in the spec/ directory to see more usage examples. Also look at the spec models in spec/spec_helper.rb.
