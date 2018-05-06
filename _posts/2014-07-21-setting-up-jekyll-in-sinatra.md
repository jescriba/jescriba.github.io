---
layout: post
title: Setting Up Jekyll in Sinatra
---

My main website was built using [Sinatra](http://www.sinatrarb.com/), but I wanted a way to generate and manage content for a blog
using the popular [Jekyll](http://jekyllrb.com) static site generator. First, I did a bit of quick digging and found some
great tutorials and themes for using Jekyll. I used [Poole](http://getpoole.com/) to provide a lot of the boiler plate code behind setting up the blog which included
some nice styling themes like [Lanyon](https://github.com/poole/lanyon).

However, I had to do some minor tweaking to get Poole and Jekyll integrated within Sinatra. Namely, Sinatra handles most
portions of the site and routing while I wanted Jekyll to handle the creation and styling of the blog portion. The main
changes I made were in the default routing of the basic jekyll skeleton. This consisted of changing the appropriate
routes in layout and in _config.yml to use the /blog/ base url. I also took a brute force route for serving the static
files by writing a build script that would copy the generated html to the /views directory of my Sinatra app and copy
the stylesheets to the default /public route that Sinatra checks.

Finally, I had to edit my main.rb file to match the appropriate url requests to the appropriate blog posts. To solve
that, I wrote a basic method that would grab the path and match it to the appropriate blog post path and copy the html
files to erb files to have Sinatra render them. I took this approach because I use [Heroku](https://www.heroku.com/) to deploy my site and heroku
does not support the use of [Rack::Sendfile](https://devcenter.heroku.com/articles/rack-sendfile). I *believe* that
Sinatra would otherwise need Sendfile to serve html directly which is why I copy the files to the erb format -
arbitrarily... Easy enough.

```ruby
get '/blog/?*' do
    get_blog(request.path)
end

def get_blog(path)
    path.gsub!('/blog', '')
    path = path + "/" + 'index'
    if File.exists?("./views/blog/" + path + '.html')
        dir_path = "./views/blog" + path
        FileUtils.cp(dir_path + '.html', dir_path + '.erb')
        erb :"blog/#{path}", :layout => false
    else
        erb :not_found
    end
end
```

