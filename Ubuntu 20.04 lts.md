# Introduction

Since Ubuntu 20.04 Focal Fossa fairly new compared to the previous Ubuntu LTS Bionic Beaver, there are the same differences when installing Jitsi with JWT support. So I decided to share a separate manual for the installation of Jitsi with JWT authentication support for Ubuntu 20.04 LTS.

Security is a very important issue if we are talking about live conferencing. As Zoom had several security issues like Room Bombing, insecurity of personal data, and encryption policies, Zoom was about to lose its reputation. Immediate actions are taken by the company to cover these security issues which was out of priority as a requirement for a very fast-growing company during — and because of — COVID-19.

Jitsi has JWT implementation to provide security for web conferencing. Basically, Jitsi rooms can be created and/or joined after a successful JWT validation.

Jitsi with JWT is a very smart and simple solution perspective to add enhanced security to your Jitsi installations. But I must say it is not easy to find accurate documentation on that even on the Jitsi Community portal. Now there are few posts about Jitsi with JWT in Jitsi Community forums. But for sake of simplicity, I made this guide sheet to save the trouble that I had to endure.

I am starting this tutorial with the understanding that the user has already setup his/her server with a domanin name. Any popular cloud service will do. I have used digital ocean for this purpose.

First ssh to your domain name. In my case it is
```
ssh root@meet.faiz-tech.com
```
meet.faiz-tech.com is my domain name.

# Install Lua

Next, copy down the below scripts. These scripts will install lu, her dependencies and prosody with fixes

```
cd &&
apt-get update -y &&
apt-get install gcc -y &&
apt-get install unzip -y &&
apt-get install lua5.2 -y &&
apt-get install liblua5.2 -y &&
apt-get install luarocks -y &&
luarocks install basexx &&
wget -c https://launchpad.net/~rael-gc/+archive/ubuntu/rvm/+files/libssl1.0.0_1.0.2n-1ubuntu5.3_amd64.deb &&
wget -c https://launchpad.net/~rael-gc/+archive/ubuntu/rvm/+files/libssl1.0-dev_1.0.2n-1ubuntu5.3_amd64.deb &&
dpkg -i libssl1.0.0_1.0.2n-1ubuntu5.3_amd64.deb &&
dpkg -i libssl1.0-dev_1.0.2n-1ubuntu5.3_amd64.deb &&
luarocks install luacrypto &&
mkdir src &&
cd src &&
luarocks download lua-cjson &&
luarocks unpack lua-cjson-2.1.0.6-1.src.rock &&
cd lua-cjson-2.1.0.6-1/lua-cjson &&
sed -i 's/lua_objlen/lua_rawlen/g' lua_cjson.c &&
sed -i 's|$(PREFIX)/include|/usr/include/lua5.2|g' Makefile &&
luarocks make &&
luarocks install luajwtjitsi &&
cd &&
wget https://prosody.im/files/prosody-debian-packages.key -O- | sudo apt-key add - &&
echo deb http://packages.prosody.im/debian $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list &&
apt-get update -y &&
apt-get upgrade -y &&
apt-get install prosody -y &&
chown root:prosody /etc/prosody/certs/localhost.key &&
chmod 644 /etc/prosody/certs/localhost.key &&
sleep 2 &&ge
shutdown -r now
```

After the instalation the server will shut down and you need to ssh again.
After getting back in the server, run the following scripts 
```

# Install jitsi-meet and jitsi-meet-tokens
cd &&
cp /etc/prosody/certs/localhost.key /etc/ssl &&
apt-get install nginx -y &&
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add - &&
sh -c "echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list" &&
apt-get -y update &&
apt-get install jitsi-meet -y &&
apt-get install jitsi-meet-tokens -y
```
Now you need to be attentive before running these scripts. The 2nd part scripts will install nginx, jitsi-meet and jitsi-meet tokens. You can use apache instead of nginx. After installing jitsi-meet, it will ask for your hostname. You need to mention your domain name which is currently hosted live.
You need to have app-id and app-secret for jwt. In my case, I used 5 randow letters for app-id and since app-sercet has to be strong, Simply run
```
hexdump -n 16 -e '4/4 "%08X" 1 "\n"' /dev/urandom
``` 
It will generate some random strings.

You also need to make sure to install ssl certification as jitsi-meet uses webrtc features. And we need TLS cerification (https) to run webrtc in our browser and server. You can generate your own certification with Let's Encrypt certificate

# Generate Let's Encrypt certificate

Simply run the following in your shell:
```
/usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
```

# OPEN FIREWALL AND PORTS
First enable ufw 
```
sudo ufw enable
```
Open ports 
```
fw allow in 22/tcp &&
ufw allow in openssh &&
ufw allow in 80/tcp &&
ufw allow in 443/tcp &&
ufw allow in 4443/tcp &&
ufw allow in 5222/tcp &&
ufw allow in 5347/tcp &&
ufw allow in 10000:20000/udp

```
You can check the status by ```sudo ufw status```

Now we need prosody authentication provider to make use of jwt token. You can read more about it [here](https://github.com/jitsi/lib-jitsi-meet/blob/master/doc/tokens.md)

Open /etc/prosody/prosody.cfg.lua and

Add above lines after admins object
```
admins = {}

component_ports = { 5347 }
component_interface = "0.0.0.0"
```
Edit
```
c2s_require_encryption=false
```
Include "conf.d/*.cfg.lua" at the end of the line
These would configure prosody.

# Configure Prosody Manual Plugin
Open  ```/etc/prosody/conf.avail/<your_domain_name>.cfg.lua```

### Under you domain config change authentication to "token" and provide application ID, secret and optionally token lifetime:

```
VirtualHost "jitmeet.example.com"
    authentication = "token";
    app_id = "<app-id that you mentioned above>";             -- application identifier
    app_secret = "<app_secret that you mentioned above>";     -- application secret known only to your token
```
### To access the data in lib-jitsi-meet you have to enable the prosody module mod_presence_identity in your config.

```
VirtualHost "<Your hostname"
    modules_enabled = { "presence_identity" }
```
### Enable room name token verification plugin in your MUC component config section:

```
Component "conference.<Your hostname>" "muc"
    modules_enabled = { "token_verification" }
```
### Setup guest domain
```
VirtualHost "guest.<Your hostname>"
    authentication = "token";
    app_id = "<app-id that you mentioned above>";
    app_secret = "<app-secret that you mentioned above>";
    c2s_require_encryption = true;
    allow_empty_token = true;
```
### Enable guest domain in config.js
Open your meet config in `/etc/jitsi/meet/<host>-config.js` and enable
```js
var config = {
    hosts: {
        ... 
        // When using authentication, domain for guest users.
        anonymousdomain: 'guest.<Your host name>',
        ...
    },
    ...
    enableUserRolesBasedOnToken: true,
    ...
}
```

### Edit jicofo sip-communicator in `/etc/jitsi/jicofo/sip-communicator.properties`
```
org.jitsi.jicofo.auth.URL=XMPP:<Your host name>
org.jitsi.jicofo.auth.DISABLE_AUTOLOGIN=true
```

### Edit jicofo config in `/etc/jitsi/jicofo/config`
SET the follow configs
```
JICOFO_HOST=<Your host name>
```

### And edit videobridge config in `/etc/jitsi/videobridge/config`

Replace 
```
JVB_HOST=
```
TO
```
JVB_HOST=<Your host name>
```

And add after `JAVA_SYS_PROPS`
```
JAVA_SYS_PROPS=...
AUTHBIND=yes
```

### Edit hostname on `/etc/hosts`
Add
```
127.0.0.1 <Your host name>
```

Then, restart all services
```bash
systemctl restart nginx prosody jicofo jitsi-videobridge2
```
And you are set. Now when you hit to your domain and create the room, it will ask for the authentication. Once authenticated you will be given the role of a moderator. In order to obtain jwt token:
```
1. go to jwt.io
2.Paste this in Payload section 
  {
  "context": {
    "user": {
      "avatar": "https:/gravatar.com/avatar/abc123",
      "name": "John Doe",
      "email": "jdoe@example.com",
      "id": "abcd:a1b2c3-d4e5f6-0abc1-23de-abcdef01fedcba"
    }
   
  },
  "aud": "jitsi",
  "iss": "<Your app-id",
  "sub": "<Your host name>",
  "room": "*"
}
3. Paste your app-secret in place of your 256-bit-secret in verify signature section
4. Copy the generated token at the left side
5. Go back to your created room and in url after your meeting room name write ?jwt=<Generated jwt tokens> 
    For exmaple - meet.faiz-tech.com/myroom?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```
Now if things turn to be correct, then you will be authenticated and become moderator or admin.
