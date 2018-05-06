---
layout: post
title: Background Jobs with Resque
---


# Background
I wanted to do a quick review of setting up background jobs for a [Sinatra](http://www.sinatrarb.com) app I have deployed with [Heroku](http://www.heroku.com). My web app serves as a music archive where I can upload songs for reference. Naturally, I wanted the site to be able to handle lossless audio formats so I could have acccess to both the lossless and lossy formats. This is where Resque comes in. [Resque](https://github.com/resque/resque) is a Redis-backed Ruby library for creating job queues, and [Redis](https://redis.io) is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker. In my example, I set up a jobs queue for transcoding my lossless audio formats to mp3 using [ffmpeg](https://ffmpeg.org/about.html). In this post, I'll show you how I got Sinatra, ffmpeg, Resque, and Redis working together in Heroku.

# RESTful Music Archive
A RESTful music archive with Sinatra that handles lossless and lossy audio formats.

Source: <https://github.com/jescriba/music-archive> 

### Setting up Resque

After installing and requiring the `resque` gem, I added some uploading logic to my route `POST` to `artists/:id/songs` that checks if the uploaded file format is of type `audio/x-aiff` among other lossless formats. 

References: [POST to artists/:id/songs](https://github.com/jescriba/music-archive/blob/a4749dd6a8e89b309a594fbf1e64fcae8d2abc52/main.rb#L353)
[Upload Song](https://github.com/jescriba/music-archive/blob/a4749dd6a8e89b309a594fbf1e64fcae8d2abc52/main.rb#L766)

To queue up the transcoding logic, I call: `Resque.enqueue(SongTranscoder, upload_params)` after the initial upload. The rest of the logic lives inside my `SongTranscoder` and `Rakefile`. Resque will automatically call the class `perform` method. In this method, I download the lossless file from s3 then shell out to ffmpeg to transcode to mp3.

```ruby
  def self.perform(upload_params)
    s3 = Aws::S3::Resource.new(region: 'us-west-1')

    # Ensure we are working w/ symbols
    upload_params = upload_params.reduce({}) { |memo, (k, v)| memo.merge({ k.to_sym => v}) }
    extension = File.extname(upload_params[:lossless_url])
    # Download s3 lossless to local
    upload_params[:tempfile_path] = "/app/tmp/temp#{extension}"
    download_cmd_str = "curl -o #{upload_params[:tempfile_path]} #{upload_params[:lossless_url]}"
    download_cmd =  `#{download_cmd_str}`
    tempfile = File.new(upload_params[:tempfile_path], "r")

    # transcode to V0 mp3
    temp_fi_dirname = File.dirname(upload_params[:tempfile_path])
    lossy_file_path = "#{temp_fi_dirname}/#{upload_params[:song_name]}.mp3"
    cmd = "ffmpeg -i #{upload_params[:tempfile_path]} -codec:a libmp3lame -qscale:a 0 #{lossy_file_path}"
    `#{cmd}`
    lossy_file = File.new(lossy_file_path, "r")

    # upload lossy
    lossy_object_path = upload_params[:lossy_url].sub(upload_params[:base_url], "")
    s3_lossy_object = s3.bucket(BUCKET).object(lossy_object_path)

    # upload
    s3_lossy_object.upload_file(lossy_file, acl: 'public-read')
    s3_lossy_object.copy_to("#{s3_lossy_object.bucket.name}/#{s3_lossy_object.key}",
                            :metadata_directive => "REPLACE",
                            :acl => "public-read",
                            :content_type => "audio/mpeg",
                            :content_disposition => "attachment; filename='#{upload_params[:song_name]}.mp3'")

    # Clean up lossy temp file
    FileUtils.rm(lossy_file_path)

    # Clean up - delete temp file
    FileUtils.rm([upload_params[:tempfile_path]])
  end
```

References: [SongTranscoder](https://github.com/jescriba/music-archive/blob/a4749dd6a8e89b309a594fbf1e64fcae8d2abc52/models/song_transcoder.rb)

**Setting up Resque Server**

Resque comes with an easy to set up server that'll be helpful for monitoring jobs.

In my config.ru file, I map the `/resque` route to the Resque server:

```ruby
require 'resque/server'
require_relative 'main'
require_relative 'globals'

$stdout.sync = true

map "/resque" do
  Resque::Server.use Rack::Auth::Basic do |username, password|
    [username, password] == [Globals::AUTH_USER, Globals::AUTH_PASSWORD]
  end

  run Resque::Server
end

map "/" do
  run Main
end
```

### Configuring Heroku

**Add Worker Dyno**

Next I added a worker dyno for running the queue of jobs from the Rakefile:

`QUEUE=* RACK_ENV=production bundle exec rake resque:work`

**Add Redis To Go**

In my `Rakefile` and `Globals`, I set up Resque per rack environment to use different redis servers: 

```ruby
if ENV["RACK_ENV"] != 'production'
  Resque.redis = 'localhost:6379'
else
  uri = URI.parse(ENV["REDISTOGO_URL"])
  Resque.redis = Redis.new(:host => uri.host, :port => uri.port, :password => uri.password)
end
```

Note: I set `REDISTOGO_URL` in my Heroku environment variables to that specified by the Heroku Redis To Go Add-On.

**Add Ffmpeg Build Pack**

Lastly, to get ffmpeg to actually work on Heroku, I added an ffmpeg [build-pack](https://github.com/jonathanong/heroku-buildpack-ffmpeg-latest.git).


