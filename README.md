Amazon S3 Ruby library that leverages [EventMachine](http://rubyeventmachine.com/) and [em-http-request](http://github.com/igrigorik/em-http-request).

By using EventMachine Happening does not block on S3 downloads/uploads thus allowing for a higher concurrency.

Happening was developed by [Peritor](http://www.peritor.com) for usage inside Nanite/EventMachine. 
Alternatives like RightAws block during the HTTP calls thus blocking the Nanite-Agent.

For now it only supports GET, PUT and DELETE operations on S3 items. The PUT operations support S3 ACLs/permissions.
Happening will handle redirects and retries on errors by default.

Installation
============

    gem install happening

Usage
=============

    require 'happening'
    
    EM.run do
      item = Happening::S3::Item.new('bucket', 'item_id')
      item.get # non-authenticated download, works only for public-read content
    
      item = Happening::S3::Item.new('bucket', 'item_id', :aws_access_key_id => 'Your-ID', :aws_secret_access_key => 'secret')
      item.get # authenticated download
      
      item.put("The new content")
      
      item.delete
    end
    
The above examples are a bit useless, as you never get any content back. 
You need to specify a callback that interacts with the http response:

    EM.run do
      item = Happening::S3::Item.new('bucket', 'item_id', :aws_access_key_id => 'Your-ID', :aws_secret_access_key => 'secret')
      item.get do |response|
        puts "the response content is: #{response.response}"
        EM.stop
      end
    end
    
This will enqueue your download and run it in the EventMachine event loop.

You can also react to errors:

    EM.run do
      on_error = Proc.new {|response| puts "An error occured: #{response.response_header.status}"; EM.stop }
      item = Happening::S3::Item.new('bucket', 'item_id', :aws_access_key_id => 'Your-ID', :aws_secret_access_key => 'secret')
      item.get(:on_error => on_error) do |response|
        puts "the response content is: #{response.response}" 
        EM.stop
      end
    end

If you don't supply an error handler yourself, Happening will be default raise an Exception.

Downloading many files could look like this:

    EM.run do
      count = 100
      on_error = Proc.new {|http| puts "An error occured: #{http.response_header.status}"; EM.stop if count <= 0}
      on_success = Proc.new {|http| puts "the response is: #{http.response}"; EM.stop if count <= 0}
      
      count.times do |i|
        item = Happening::S3::Item.new('bucket', "item_#{i}", :aws_access_key_id => 'Your-ID', :aws_secret_access_key => 'secret')
        item.get(:on_success => on_success, :on_error => on_error)
      end
    end
    
Upload
=============

Happening supports the simple S3 PUT upload:    
  
    EM.run do
      on_error = Proc.new {|http| puts "An error occured: #{http.response_header.status}"; EM.stop }
      item = Happening::S3::Item.new('bucket', 'item_id', :aws_access_key_id => 'Your-ID', :aws_secret_access_key => 'secret', :on_success => on_success, :on_error => on_error)
      item.put( File.read('/etc/passwd'), :on_error => on_error ) do |response|
        puts "Upload finished!"; EM.stop 
      end
    end
    
Setting permissions looks like this:

    EM.run do
      on_error = Proc.new {|http| puts "An error occured: #{http.response_header.status}"; EM.stop }
      on_success = Proc.new {|http| puts "the response is: #{http.response}"; EM.stop }
      item = Happening::S3::Item.new('bucket', 'item_id', :aws_access_key_id => 'Your-ID', :aws_secret_access_key => 'secret', :permissions => 'public-write')
      item.get(:on_success => on_success, :on_error => on_error)
    end

Deleting
=============

Happening support the simple S3 PUT upload:    
  
    EM.run do
      on_error = Proc.new {|response| puts "An error occured: #{response.response_header.status}"; EM.stop }
      item = Happening::S3::Item.new('bucket', 'item_id', :aws_access_key_id => 'Your-ID', :aws_secret_access_key => 'secret')
      item.delete(:on_error => on_error) do |response|
        puts "Deleted!"
        EM.stop
      end
    end

Amazon returns no content on delete, so having a success handler is usually not needed for delete operations.

Credits
=============

The AWS signing and canonical request description is based on [RightAws](http://github.com/rightscale/right_aws).
    
    
License
=============

Happening is licensed under the OpenBSD / two-clause BSD license, modeled after the ISC license. See LICENSE.txt


About
=============

Happening was written by [Jonathan Weiss](http://twitter.com/jweiss) for [Peritor](http://www.peritor.com).
