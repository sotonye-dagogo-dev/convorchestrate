FROM node:20-alpine AS builder
RUN apk add --no-cache python3 make g++
WORKDIR /app
COPY package.json package-lock.json turbo.json tsconfig.base.json ./
COPY apps/dashboard/package.json apps/dashboard/package.json
COPY packages/core/package.json packages/core/package.json
COPY packages/meta-api/package.json packages/meta-api/package.json
COPY packages/schemas/package.json packages/schemas/package.json
COPY packages/utils/package.json packages/utils/package.json
RUN npm install
COPY . .
RUN npx turbo run build --filter=dashboard...

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/apps/dashboard/.next/standalone ./
COPY --from=builder /app/apps/dashboard/.next/static ./apps/dashboard/.next/static
EXPOSE 3000
ENV PORT=3000
CMD ["node", "apps/dashboard/server.js"]
