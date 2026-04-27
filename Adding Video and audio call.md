## Adding Video and Audio Calling feature


You can use a standard coTURN image to facilitate the service 

```sh 
services:
  coturn:
    image: coturn/coturn
    container_name: coturn
    restart: unless-stopped
    ports:
      - "3478:3478/tcp"
      - "3478:3478/udp"
      - "49152-65535:49152-65535/udp" # Relay ports range
    environment:
      - COTURN_REALM=://domain.com
      - COTURN_STATIC_AUTH_SECRET=your_secret_here
    # Alternatively, use host networking for better NAT traversal
    # network_mode: host 

```

then add the information on nextcloud container with enviroment variable

```sh

  nextcloud:
    image: nextcloud:latest
    environment:
      - TURN_DOMAIN=turn.example.com
      - TALK_PORT=3478
      - TURN_SECRET=your_generated_secret
      # Using the NC_ prefix for generic config.php overrides
      - NC_talk_stun_servers=["://example.com"]
```
