---
layout: post
title: Smart Curtains and ntfy
date: 2025-08-01 14:00 +0100
summary: |
  An adventure in socket programming with Ruby, ntfy, and smart curtains.
---

We recently installed smart curtains in our home.
They are controlled with a mobile app via either Bluetooth (directly) or Wi-Fi (via a bridge).
I have installed the Wi-Fi bridge and it works well, but we also have smart switches that control all of our lighting.
These Matrix switches have four buttons each, and, since we're not using all four possibilities on each switch, I thought it would be nice to use one of the buttons to control the curtains.

I thought that since they were accessible via Wi-Fi, I could just send a request to the bridge to open or close the curtains.
However, the bridge is not a web server, so I had to do some digging.

After some research, I found that the original manufacturer of the curtains, not the white label I've bought them under, had some documentation available online.
It turns out that the bridge speaks via UDP, so I could presumably send a UDP packet to the bridge to control the curtains.

It was done cleverly enough, so you needed an access token to send commands, luckily, this was also documented, and a bit odd to obtain.
In their app, you could clikc thrice on the about page, and the token would be displayed in a dialog.

I did this, used to documentation to figure out the UDP packet format, and wrote a small Ruby script to send the commands.

The format is simple enough, you send a packet with the following format:

```ruby
{
  AccessToken: ACCESS_TOKEN,
  mac: curtainMacAddress,
  msgType: "WriteDevice",
  deviceType: "10000000", # Always 1000000 for curtains
  msgID: Time.now.strftime("%Y%m%d%H%M%L"), # Unique ID for the message
  data: {
    operation: op # 0 is close, 1 is open, 2 is stop, 5 is status
  }
}
```

With this in hand, I created a small Sinatra app that could run on my local network, listening for HTTP requests to open or close the curtains.
Our Matrix switches used Fibaro Home Center to control the lights, and you can create quick applications and scenes in Home Center to do your custom logic.
I created a scene that would send an HTTP request to my Sinatra app to open or close the curtains when a button was pressed.

The Sinatra app is very simple, it listens for requests with the mac address and operation in the query string, and sends the UDP packet to the bridge.

```ruby
send_socket = UDPSocket.new
data = query(@mac_address, 1) # Creates the JSON object above

begin
  Timeout.timeout(3) do
    resp, _addr = send_socket.recvfrom(4096)
    json = JSON.parse(resp)
    ... # do stuff with the response
  rescue Timeout::Error
    # No response for this curtain
  end
ensure
  send_socket.close
end
```

I also wanted to be able to see the battery status of the curtains, so I added a route to the Sinatra app that would send a status request to the bridge.

This resulted in a simple dashboard that I can access from my phone.

<div class="flex justify-center">
  <img src="/assets/posts/2025-08-01-dashboard.png" alt="Dashboard">
</div>

When a battery goes below 20%, the Home Center sends a request to my local instance of ntfy, which then sends a notification to my phone.

Since the Matrix buttons also have integrated LEDs, I can also use them to indicate the battery status of the curtains, so the controlling button flashes red between 9 am and 10 am while the battery is low.

I'm looking forward to adding more features to this setup, like scheduling the curtains to open and close at specific times, or integrating them with other smart home devices.
