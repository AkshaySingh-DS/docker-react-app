# Stage1: Build Stage
FROM node:22-slim AS builder
ARG VITE_SERVER_URL=http://127.0.0.1:7999
ENV VITE_SERVER_URL=$VITE_SERVER_URL 
WORKDIR /client
COPY . .
RUN npm install && npm run build

#Stage2: Runtime Stage
FROM node:22-slim 
WORKDIR /client

#copy only the necessary files from build stage
COPY --from=builder /client/public/ ./public
COPY --from=builder /client/src/ ./src
COPY --from=builder /client/index.html ./index.html
COPY --from=builder /client/package.json  ./package.json
COPY --from=builder /client/package-lock.json  ./package-lock.json
COPY --from=builder /client/vite.config.js ./vite.config.js
COPY --from=builder /client/node_modules ./node_modules
COPY --from=builder /client/dist ./dist

# Command to run when the container starts
CMD ["bash", "-c", "npm run preview"]