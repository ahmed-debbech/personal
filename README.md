### Personal site

1 - site calls blog when you hit `debbech.com/blog` with apache reverse proxy rule in `site/conf/httpd.conf` \
2 - hugo blog is set `debbech.com/blog` as base URL to point all blog traffic to the reverse proxy config in apache \
3 - `site/conf` directory contains config for apache \
4 -  `posts` dir in host is synchronous with blog docker instance for blog content

##### Run it with:
1 - make sure to set HOST variable in `docker-compose.yml` to the blog host \
2 - `docker compose up -d --build` will spin up one local port listening on 2900 for both site and blog.