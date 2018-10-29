FROM node:8-alpine

# install python
RUN apk add --no-cache curl python

RUN curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip" && \
  unzip awscli-bundle.zip && \
  ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws

RUN /usr/local/bin/aws --version

CMD ["/bin/sh"]
