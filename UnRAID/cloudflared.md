# Introduction
When setting up my containers on UnRAID so that I can hook them up to cloudflared I put them into different networks, this way I am able to connect them by name.
# Initial config (Cloudflare)
## Domain setup
After login to cloudflare you will need to add a domain/site
![[CloudFlare_add_site.png]]
![[CloudFlare_plan.png]]
Cloudflare will then scan for DNS records and you should have something similar to this
![[CloudFlare_dns.png]]
You will then be instructed to change your nameservers for this domain
![[CloudFlare_nameservers.png]]
As it states this could take about 24 hours to update on your registra, I have been fortunate that this took a couple of minutes.
We then get the option to configure some security and optimizations
![[CloudFlare_QuickStart.png]]
I recomend switching all options on
![[CloudFlare_QuickStart_done.png]]
You will then get shown the following page, where you can check the nameservers to see if the new NS settings have propogated
![[CloudFlare_Overview.png]]
## Tunnel setup (Zero Trust)
On the cloudflare main dashboard there is a link to ![[CloudFlare_ZeroTrust_link.png]]
You can then go to Access > Tunnels
And `Create a tunnel`
![[CloudFlare_Tunnel_create.png]]
Under environment choose `Docker` #tunnel
![[CloudFlare_configure_tunnel.png]]
This will give you a command to run, you will need to save this command for later, we use parts of it when  we create the cloudflared image in UnRAID.
Then we setup our first `route` replacing unraid_ip with the IP of your unraid server (this could be the internal docker ip `172.17.0.1`)
![[CloudFlare_tunnel_route.png]]
Don't worry about the DNS warning as it will create this record for you.
Under your tunnel list you should now see the new tunnel
![[CloudFlare_tunnel_list.png]]
## Access control
Under the access menu there is a `Applications` section, click on this and then `Add an application`
![[CloudFlare_Application_config.png]]
Choose `Self-Hosted` 
![[CloudFlare_ZeroTrust_configure_app.png]]
Then we need to setup a policy
![[CloudFlare_ZeroTrust_add_application.png]]
![[CloudFlare_ZeroTrust_create_rules.png]]
All other options I leave as default.
This should now ask for an email address to send a token to when ever someone goes to that url `unraid.cloudflaretest.co.za` is this example, Cloudflare will then send you an email with a code for you to enter, and then grant you access to the site. (once you have finished the setup of the tunnel on UnRAID)
# Initial config (UnRAID)
The first thing we do is make sure that the custom networks get preserved.
Navigate to: Settings -> Docker
Make sure that: Preserve user defined networks: `Yes`

## Docker Networks
To create a new docker network you will need to go to the console (or ssh)
![[UnRAID_console_button.png]]
We can then create a new network
```bash
root@tower:~# docker network create cloudflare_network
ab71326d697d9145b6de17d8b15c8c6ca8480ea572736e0a67eb6dccfbcc0960
```
We can then check the list of networks
```bash
root@medusa:~# docker network ls
NETWORK ID     NAME                 DRIVER    SCOPE
05c520607633   br0                  macvlan   local
caf701783d35   bridge               bridge    local
ab71326d697d   cloudflare_network   bridge    local
9972a11cbad4   host                 host      local
5558164e32d0   none                 null      local
```
The network ID's will be different on each server
## Cloudflared
I am using the official cloudflared image
![[UnRAID_cloudflared_app.png]]
When adding this `App` change to advanced view, here we will change the following options:
`Network Type` : `cloudflare_network`
`Post Arguments`: `tunnel --no-autoupdate run --token <<token>>`
(This token can be found in the docker run command you saved from ealier) [Tunnel Setup Zero Trust](#tunnel-setup-zero-trust)
## Apps (Docker containers)
For each application you want to have exposed via cloudflare we need to add it to this new network, as can be seen by the `Network Type` field

For this example I am using ApacheGuacamole, for other applications this name would be replaced with the name of the container
![[UnRAID_Guacamole_settings.png]]
The other way to add this container to a network requires switching to Advanced view and adding the following to the `Post Arguments` field: `&& docker network connect cloudflare_network ApacheGuacamole`
## Cloudflare tunnel
Now we can create an entry in the tunnel for this new application.
It will be the same as the unraid one we created earlier
![[CloudFlare_tunnel_route.png]]
The only difference is we will give it a new subdomain and the URL will be `ApacheGuacamole:8080`
![[UnRAID_Guacamole_port.png]]
As you will see for the cloudflared tunnel to pass through we need to use the container port `8080` and not the exposed port `8888` as the traffic is sent across the docker network
We then follow the [Access control](#access-control) steps to add an extra layer of security (if you choose)