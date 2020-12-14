FROM golang as build-stage
WORKDIR /app

COPY go.mod /app
COPY go.sum /app
RUN go mod download

COPY main.go /app

RUN cd /app && GOOS=linux go build -o weather-data-gen

RUN ls -lrt
CMD ["/app/weather-data-gen"]