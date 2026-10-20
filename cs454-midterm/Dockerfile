FROM node:22

WORKDIR /app

COPY --chown=node:node package.json ./
COPY --chown=node:node src ./src

ENV PORT=8080

EXPOSE 8080

USER node

CMD ["npm", "start"]