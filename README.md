# Nginx Configuration For Rails

This page describes configuration (example) of the Nginx with Rails (through Unicorn).

**/etc/nginx/nginx.conf**

    worker_processes 8;

    user nobody nobody;

    events {
      worker_connections 4096;
      accept_mutex on;
      use epoll;
    }

    http {
      proxy_buffer_size 128k;
      proxy_buffers 4 256k;
      proxy_busy_buffers_size 256k;

      client_max_body_size 40M;

      include mime.types;
      default_type application/octet-stream;

      sendfile on;

      gzip on;
      gzip_vary on;
      gzip_proxied any;
      gzip_min_length 500;
      gzip_http_version 1.0;
      gzip_disable "MSIE [1-6]\.";
      gzip_types text/plain text/xml text/css text/javascript application/x-javascript application/xml application/json;

      server {
        return 404;
      }

      include /etc/nginx/servers/*;
    }

**/etc/nginx/servers/example.conf**

    server {
      listen 80;
      server_name dsperansky.info;
      root /var/www/dsperansky.info/public;

      location ~* ^/assets/ { expires 1d; }

      location / {
        try_files $uri @unicorn;
      }

      location @unicorn {
        proxy_pass http://unix:/var/www/dsperansky.info/tmp/sockets/unicorn.sock;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
      }
    }

**config/unicorn.rb**

    worker_processes 5

    dir = '/var/www/dlibrary.org/vsp/'

    working_directory dir

    preload_app true

    listen dir + 'tmp/sockets/unicorn.sock', :backlog => 64

    timeout 60

    pid dir + 'tmp/pids/unicorn.pid'

    stderr_path dir + 'log/unicorn.stderr.log'
    stdout_path dir + 'log/unicorn.stdout.log'

    before_fork do |server, worker|
      defined?(ActiveRecord::Base) and ActiveRecord::Base.connection.disconnect!
    end

    after_fork do |server, worker|
      defined?(ActiveRecord::Base) and ActiveRecord::Base.establish_connection
    end

**start**

    test -s "tmp/pids/unicorn.pid" && kill -QUIT `cat tmp/pids/unicorn.pid`

**stop**

    bundle exec unicorn -c config/unicorn.rb -E production -D'

