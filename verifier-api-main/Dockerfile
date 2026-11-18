# ---- base (Node 22 + pnpm + Chromium) ----
FROM node:22-bookworm-slim AS base
WORKDIR /app

# Copy standalone pnpm binary from official pnpm image
COPY --from=ghcr.io/pnpm/pnpm:10 /opt/pnpm /opt/pnpm
RUN for bin in pnpm pn pnpx pnx; do ln -s "/opt/pnpm/$bin" "/usr/local/bin/$bin"; done

# Skip Puppeteer's bundled Chromium download (we install system chromium below)
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true \
    PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium

# System dependencies for Puppeteer / Chromium
RUN apt-get update && apt-get install -y --no-install-recommends \
    libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 libgbm1 libasound2 \
    libpangocairo-1.0-0 libxss1 libgtk-3-0 libxshmfence1 libglu1 chromium curl wget ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY pnpm-lock.yaml package.json pnpm-workspace.yaml* ./
COPY prisma ./prisma

# ---- deps (install devDeps) ----
FROM base AS deps
RUN --mount=type=cache,target=/root/.local/share/pnpm/store/v3 \
    pnpm install --frozen-lockfile --prod=false

# ---- build ----
FROM deps AS build
COPY . .
RUN pnpm prisma generate && pnpm build
RUN pnpm prune --prod

# ---- runtime ----
FROM base AS runtime
ENV NODE_ENV=production

# Copy built artifacts and production dependencies from build stage
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./package.json
COPY --from=build /app/prisma ./prisma
COPY --from=build /app/eng.traineddata ./eng.traineddata

# Run database migrations automatically on startup, then start the API
CMD ["sh", "-c", "pnpm prisma migrate deploy && node dist/index.js"]
