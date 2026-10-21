FROM node:24-slim

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# Địa chỉ backend mà TRÌNH DUYỆT gọi, được nhúng vào mã lúc build
ARG NEXT_PUBLIC_API_URL=http://localhost:3001
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL

RUN npm run build

ENV NODE_ENV=production
ENV HOSTNAME=0.0.0.0
EXPOSE 3000

CMD ["npm", "run", "start"]