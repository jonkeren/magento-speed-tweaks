# Manual for basic Magento2 speedup tweaks.
These are instructions for Hypernode. Instructions for Hipex would be a bit different. 
Jos



### 1. varnish vcl:

1. upload [the tweaked vcl file](https://github.com/JosQlicks/magento-speed-tweaks/blob/main/vcl-jhp-optimized-jos.vcl) to `/data/web/magento2`
2. `varnishadm vcl.load mag2 /data/web/magento2/vcl-jhp-optimized-jos.vcl`
3. `varnishadm vcl.use mag2`
4. `varnishadm vcl.list`  (see if new varnish vcl is used)
5. wait 5-6 minutes (see timestamp on `/data/var/varnish/default.vcl` <-- this needs to be updated)
6. `hypernode-servicectl restart varnish`
7. `hypernode-servicectl reload nginx`
8. check `varnishstat` for hit rate and cache usage.



### 2. nginx buffers:

1. Create extra file in: `/data/web/nginx` --> filename: `server.header_buffer_jos`
2. add content:
```
fastcgi_buffers 16 16k;
fastcgi_buffer_size 32k;
proxy_buffer_size 128k;
proxy_buffers 4 256k;
proxy_busy_buffers_size 256k;
```
3. save
4. `hypernode-servicectl reload nginx`



### 3. Block bots (especially bingbot; GTFO):

1. Create extra file in: `/data/web/nginx` --> filename: `server.bots_goaway_jos`
2. add content:
```
if ($http_user_agent ~* (360Spider|bingbot|BLEXbot|SEOKicks|Mauibot|Riddler|ltx71|ZoominfoBot|seznam|velen|GrapeshotCrawler|Baidu|Censys|Pinterest) ) {
    return 410;
}
```
3. save
4. `hypernode-servicectl reload nginx`




### 4. Optimize jpg images losslessly on server:
in the `/data/web/magento2/pub/media/` directory.
(except `/catalog` dir because of SRS import). This also makes jpgs load progressively in browsers.

1. `cd /data/web/magento2/pub/media/`
2. `find . -type f -not -path "./catalog/*" -not -path "./tmp/*" -not -path "./import/*" -name "*.jpg" -exec jpegoptim --all-progressive -p -t -v -P {} \;`
3. let this run in the background. Can take a long time.




### 5. magento backend settings:
1. stores > configuration > mirasvit > page cache warmer:
--> "Forcibly make pages cacheable"; set: configured + set 3 checkboxes; save config.

2. stores > configuration > advanced > system > full page cache: 
--> set TTL for public content to: 2629743 (1 month); save config.


### 6. Preconnect external domains in the HTTP header
This pre-connects the DNS to domains that will be loaded later, after the HTML has parsed. This speeds up page loads with slow-loading external sites/scripts, by already connecting to those domains AS SOON AS the http header is sent to the browser.
Longer explanation: https://andydavies.me/blog/2019/03/22/improving-perceived-performance-with-a-link-rel-equals-preconnect-http-header/

Change the nginx config:

1. Create extra file in: `/data/web/nginx` --> filename: `server.preconnect.conf`
2. add content (set domains etc. accordingly)
```
add_header Link '<https://cloud.wordlift.io/>; rel=preconnect; crossorigin=anomynous; probability=1.0;';
add_header Link '<https://connect.nosto.com/>; rel=preconnect; crossorigin=anonimous; probability=1.0;';
add_header Link '<https://cloud.wordlift.io/app/bootstrap.js>; as=script; crossorigin=anonymous; rel=preload;';
```
3. save
4. `hypernode-servicectl reload nginx`
Done.
Test the header by using `curl -I https://www.example.com`.


====


## Improvements general:
- Upgrade Magento from 2.x naar 2.4.x, including ElasticSearch. ("Search_tmp table" bug); lower mysql load.
- Upgrade Template (Theme) to newest version.
- Improve SRS import script: faster -> import only changed items; 
- flush *only* cache for changed items, not complete FPC
- Remove unneeded JS and external pixels/scripts from template
- Configure and use Cloudflare.

