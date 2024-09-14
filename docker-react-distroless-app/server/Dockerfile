# Stage1: Build Stage
FROM node:22-slim AS builder
WORKDIR /server
COPY . .
RUN npm install && npx prisma generate

#Stage2: Runtime Stage
FROM node:22-slim
WORKDIR /server

#copy only the necessary files from build stage
COPY --from=builder /server/node_modules ./node_modules
COPY --from=builder /server/prisma/ ./prisma
COPY --from=builder /server/package.json ./package.json
COPY --from=builder /server/src/ ./src
COPY --from=builder /server/package-lock.json ./package-lock.json

ENV DATABASE_URL="postgresql://john_doe:john.doe@database:5432/docker_test_db?schema=public"
ENV PORT=8000

#prisma required below dependecy comes by default with node bigger image
# RUN apt-get update -y && apt-get install -y openssl 
RUN apk update && apk upgrade && apk add --no-cache \
    openssl

EXPOSE 8000
# Command to run when the container starts
CMD ["bash", "-c", "npx prisma migrate reset --force && npm start"]