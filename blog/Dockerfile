FROM alpine

WORKDIR /app

RUN wget https://github.com/gohugoio/hugo/releases/download/v0.157.0/hugo_0.157.0_linux-amd64.tar.gz

RUN apk add tar

RUN tar -xvf hugo_0.157.0_linux-amd64.tar.gz

COPY . .

ARG HOST
ENV HOST_ENV=$HOST

CMD sh -c "./hugo --bind=0.0.0.0 serve -D --baseURL=http://$HOST_ENV/blog --appendPort=false"