FROM node:18
WORKDIR /server
COPY src/ /server/src
COPY prisma/ /server/prisma
COPY package.json /server

EXPOSE 8000

ENV DATABASE_URL="postgresql://john_doe:john.doe@database:5432/docker_test_db?schema=public"
ENV PORT=8000

RUN npm install
RUN npx prisma generate

# Command to run when the container starts
CMD ["bash", "-c", "npx prisma migrate reset --force && npm start"]