
Implementing Django-Socketio
============================


To begin, I'm going to assume you already have a Django server environment set up. If you don't, you can follow these instructions `here <http://aliteralmind.wordpress.com/2014/08/07/doingthedeepdowndiggitydivewithdjangoanddigitalocean>`_ , which I've found to be generally pretty good. It should be noted that these instructions assume you are using a DigitalOcean droplet, but an AWS instance running Ubuntu 14.04 will work just as well

The first thing you want to do to get django-socketio up and running is install git:
::

  sudo apt-get install git

This will allow you to clone the repository. If you have pip installed inside your virtualenv, activate the virtualenv first.::

  pip install -e git+git://github.com/IsaacRay/django-socketio.git#egg=django-socketio

Now you have django-socketio installed. Next, you have to set up nginx to proxy to your SocketIO server, which by default runs on port 9000. This is my nginx config to make this work:
::
    
          server {
              listen 80;
                  server_name example.com;

                  access_log off;

                  location /static/ {
                      alias /opt/myenv/static/;
                  }

                  location / {
                      proxy_pass http://127.0.0.1:8001;
                      proxy_set_header X-Forwarded-Host $server_name;
                      proxy_set_header X-Real-IP $remote_addr;
                      add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
                  }
                  location /socket.io/ {
                      proxy_pass http://127.0.0.1:9000;
                      proxy_set_header Upgrade $http_upgrade;
                      proxy_set_header Connection "upgrade";
                      proxy_http_version 1.1;

                  }
              }
          
The key part here is the location block catching requests for /socket.io/. This block takes care of everything you need to have your sockets proxied to the SocketIO server.
Once you have this set up, the next parts are pretty easy. You need to make an events.py in your Django app. Here is an example of mine:
::

          from socketio.namespace import BaseNamespace
          from django_socketio.events import Namespace
          from socketio.mixins import BroadcastMixin

          @Namespace('/echo')
          class EchoNamespace(BaseNamespace, BroadcastMixin):
              nicknames = []

              def initialize(self):
                  pass
              
              def on_echo(self, echo):
                  self.broadcast_event('echo', echo)

              def recv_disconnect(self):
                  self.log('Disconnected')
                  self.disconnect(silent=True)
                  return True
        
This is a very basic events.py, but the crucial things are here. We have a namespace decorator, which registers the namespace "/echo" with the SocketIO server. We have an on_echo function, which will get called when the socket triggers an "echo" event, which we'll get to in a moment. And we have a recv_disconnect, which gets called when the socket disconnects. There is also an intialize function which you can use to set things up when the socket first connects. I typically haven't used it, but I've included it for reference.
Now lets look at the template. In the template you need to include the socketio JS. You can do that like this:

.. code-block:: html

  <script type="text/javascript" src="//cdnjs.cloudflare.com/ajax/libs/socket.io/0.9.16/socket.io.min.js"></script>
  Next, you need to connect to your socket:
  <script>
  var socket = io.connect("/echo");
  </script>
  Lastly, lets register an echo listener:
  <script> 
  socket.on("echo", echo);

  function echo(data){
  alert(data);
  }
  </script>
        
So whats going on here? Well, io.connect() basically creates a direct tunnel to your events.py file on your server. It finds the Namespace based on what you give it, and then connects directly to a Namespace object it creates upon connection. Important Note: "/echo" is NOT a URL. This is just the way that namespace notation is written. Do not get confused by this. When you do socket.on("echo", echo);, you are saying, when you recieve a message from the server with an event type of "echo", call my echo function in the javascript. Our echo function is just going to spit out the message from the server.
The last step is actually turning on your SocketIO server. Django-socketio comes with a built in management command to do that for you:
::

  python manage.py runserver_socketio

By default this will set the server running on port 9000, where we've already told nginx to forward our websocket requests. I suggest setting this up to run automatically using Supervisor or some other process manager.

So what can we do with all this? Well, once you've got everything in place, you can navigate to your template, and pull up a developer console in your browser. Type socket.emit("echo", "hello world"); If you've done everything right, you should see an alert box with "Hello World" appear. Why is this useful? Because what happened here is you told your socket, which is connected to your events.py, to emit an "echo" event to the server. The server picks up that event and triggers the on_echo function on the Namespace instance. on_echo takes the data sent along with the event (the string "Hello World") and broadcasts it out to all the sockets that are currently connected to the namespace. Note that I said "all the sockets that are currently connected". This is where it gets cool. Go to another device, either your phone or another computer, and pull up your template. No go back to the original device, and execute the emit command again. You should see a "Hello World" alert pop up on BOTH browsers.

Thats the basics of implementing Websockets on Django. For more information, you can check out the docs on gevent-socketio on which django-socketio is based. It will give you a little more information on Namespaces, and Mixins you can use to enhance your project. Happy Hacking!






















