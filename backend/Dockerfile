FROM node:24-slim

# Prisma cần openssl để chạy migrate
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends openssl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# Sinh Prisma Client (không cần kết nối database ở bước này) rồi build
RUN DATABASE_URL="postgresql://build:build@localhost:5432/build" npx prisma generate
RUN npm run build

ENV NODE_ENV=production
EXPOSE 3001

# Tạo bảng, nạp dữ liệu mẫu rồi chạy server
CMD ["sh", "-c", "npx prisma migrate deploy && npx prisma db seed && node dist/main.js"]