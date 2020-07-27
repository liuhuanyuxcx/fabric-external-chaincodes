# This image is a microservice in golang for the Degree chaincode
FROM golang:1.14.4-alpine3.12 AS build

COPY ./ /go/src/github.com/e2e
WORKDIR /go/src/github.com/e2e

# Build application
RUN go env -w GOPROXY=https://goproxy.io,direct && go build -o chaincode -v .

# Production ready image
# Pass the binary to the prod image
FROM alpine:3.12 as prod

COPY --from=build /go/src/github.com/e2e/chaincode /app/chaincode

USER 1000

WORKDIR /app
CMD ./chaincode
