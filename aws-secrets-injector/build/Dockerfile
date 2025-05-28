# -------------------------------------------------------------------------------------
#
# Copyright (c) 2025, WSO2 LLC. (http://www.wso2.com).
#
# This software is the property of WSO2 LLC. and its suppliers, if any.
# Dissemination of any information or reproduction of any material contained
# herein in any form is strictly forbidden, unless permitted by WSO2 expressly.
# You may not alter or remove any copyright or other notice from copies of this content.
#
# --------------------------------------------------------------------------------------

FROM golang:1.22-alpine AS build

# Set the working directory inside the container & copy go.mod and go.sum files to enable dependency caching
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

# Copy the application source code into the container & build the Go application
COPY ./application ./application
RUN go build -ldflags="-w -s" -o /app -v ./application/secret-injector

# Use a minimal runtime image & copy the compiled Go binary
FROM gcr.io/distroless/static-debian12
COPY --from=build /app /app
USER nonroot:nonroot

# Define the entrypoint to run the application
ENTRYPOINT ["/app"]
