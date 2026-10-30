FROM alpine:latest

RUN apk add --no-cache curl

CMD ["sh", "-c", "if curl -s -f http://backend:5000/health > /dev/null; then echo 'backend: OK'; else echo 'backend: UNREACHABLE'; fi"]