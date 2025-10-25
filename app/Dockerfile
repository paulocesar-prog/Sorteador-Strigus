FROM cgr.dev/chainguard/python:latest-dev AS python-builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM cgr.dev/chainguard/redis:latest AS redis-builder
WORKDIR /app

FROM cgr.dev/chainguard/go:latest AS go-builder
WORKDIR /app
COPY ./go-bin-exec/main.go .
RUN go mod init go-bin-exec && go build -o go-bin-exec .

FROM cgr.dev/chainguard/python:latest
WORKDIR /opt/app
ENV REDIS_HOST=localhost
COPY app.py .
COPY static/ static/
COPY templates/ templates/
COPY --from=python-builder /home/nonroot/.local /home/nonroot/.local
COPY --from=redis-builder /usr/bin/redis-server /usr/bin/redis-server
COPY --from=go-builder /app/go-bin-exec .
ENV PATH=$PATH:/home/nonroot/.local/bin
EXPOSE 5000
ENTRYPOINT ["./go-bin-exec"]
