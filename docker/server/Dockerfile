#
# conductor:server - Build Conductor Server
#
# ===========================================================================================================
# 0. Builder stage 1
# ===========================================================================================================
FROM openjdk:17-bullseye AS builder

LABEL maintainer="Orkes OSS <oss@orkes.io>"

# Copy the project directly onto the image
COPY --from=project . /conductor
WORKDIR /conductor

# Build the server on run
RUN ./gradlew build -x test
WORKDIR /server/build/libs
RUN ls -ltr


#
# conductor:server - Build Conductor UI
#
# ===========================================================================================================
# 1. Builder stage 2
# ===========================================================================================================
FROM alpine:3.20 AS ui-builder

LABEL maintainer="Orkes OSS <oss@orkes.io>"

USER root

# Install dependencies
RUN apk add --update nodejs npm yarn

COPY --from=project . /conductor
WORKDIR /conductor/ui

# Include monaco sources into bundle (instead of using CDN)
ENV REACT_APP_MONACO_EDITOR_USING_CDN=false
RUN yarn install && cp -r node_modules/monaco-editor public/ && yarn build
RUN ls -ltr
RUN echo "Done building UI"


# ===========================================================================================================
# 2. Bin stage
# ===========================================================================================================
FROM alpine:3.19

LABEL maintainer="Orkes OSS <oss@orkes.io>"

USER root

RUN apk add openjdk17
RUN apk add nginx

# Make app folders
RUN mkdir -p /app/config /app/logs /app/libs

# Copy the compiled output to new image
COPY --from=project docker/server/bin /app
COPY --from=project docker/server/config /app/config
COPY --from=builder /conductor/server/build/libs/*boot*.jar /app/libs/conductor-server.jar

# Copy compiled UI assets to nginx www directory
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY --from=ui-builder /conductor/ui/build .
COPY --from=ui-builder /conductor/docker/server/nginx/nginx.conf /etc/nginx/http.d/default.conf

# Copy the files for the server into the app folders
RUN chmod -R 775 /app
RUN chmod -R 775 /etc/nginx
RUN chmod -R 775 /var/lib/nginx
RUN chmod -R 775 /var/log/nginx
RUN chmod -R 775 /run/nginx
RUN chmod +x /app/startup.sh

HEALTHCHECK --interval=60s --timeout=30s --retries=10 CMD curl -I -XGET http://localhost:8080/health || exit 1

RUN adduser -S orkes -u 1234 -G root
RUN adduser orkes nginx
USER orkes

CMD [ "/app/startup.sh" ]
ENTRYPOINT [ "/bin/sh"]
