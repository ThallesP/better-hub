# ── Stage 1: Install dependencies ──────────────────────────────────────
FROM oven/bun:1.3.5-alpine AS deps

WORKDIR /app

# Copy workspace root manifests
COPY package.json bun.lock bunfig.toml ./

# Copy workspace package.json files for dependency resolution
COPY apps/web/package.json apps/web/package.json

# Install all dependencies, skip postinstall (prisma generate needs schema files)
RUN bun install --ignore-scripts

# ── Stage 2: Build the Next.js app ────────────────────────────────────
FROM oven/bun:1.3.5-alpine AS builder

WORKDIR /app

# Copy deps from previous stage (hoisted to root node_modules)
COPY --from=deps /app/node_modules ./node_modules

# Copy all source files
COPY . .

WORKDIR /app/apps/web

# Generate Prisma client
RUN bunx prisma generate

# Build Next.js (needs DATABASE_URL placeholder for prisma)
ENV DATABASE_URL="postgresql://placeholder:placeholder@localhost:5432/placeholder"
ENV NEXT_TELEMETRY_DISABLED=1
RUN bun run --bun next build

# ── Stage 3: Production runner ─────────────────────────────────────────
FROM node:22-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Create non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy standalone output
COPY --from=builder /app/apps/web/.next/standalone ./
COPY --from=builder /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=builder /app/apps/web/public ./apps/web/public

# Copy Prisma generated client into standalone
COPY --from=builder /app/apps/web/src/generated ./apps/web/src/generated

# Copy full node_modules + Prisma files for pre-deploy migrations
# (prisma CLI has a deep dependency tree, so copying all node_modules is simplest)
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/apps/web/prisma ./apps/web/prisma
COPY --from=builder /app/apps/web/prisma.config.ts ./apps/web/prisma.config.ts

USER nextjs

EXPOSE 3000

CMD ["node", "apps/web/server.js"]
