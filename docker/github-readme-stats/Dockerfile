# syntax=docker/dockerfile:1.7

ARG NODE_VERSION=24-alpine

FROM node:${NODE_VERSION} AS build

RUN apk add --no-cache git

WORKDIR /src

ARG GITHUB_README_STATS_REPO=https://github.com/anuraghazra/github-readme-stats.git
ARG GITHUB_README_STATS_REF=master

RUN git clone --filter=blob:none "${GITHUB_README_STATS_REPO}" . \
  && git checkout "${GITHUB_README_STATS_REF}"

# The upstream Express server is documented for non-Vercel hosting, but express
# is kept as a devDependency upstream. Promote it before installing production deps.
RUN node -e "const fs=require('fs');const pkg=JSON.parse(fs.readFileSync('package.json','utf8'));pkg.dependencies={...(pkg.dependencies||{})};if(pkg.devDependencies?.express){pkg.dependencies.express=pkg.devDependencies.express;delete pkg.devDependencies.express;}pkg.scripts={...(pkg.scripts||{}),start:'node express.js'};fs.writeFileSync('package.json',JSON.stringify(pkg,null,2)+'\n');"

RUN npm install --package-lock-only --ignore-scripts \
  && npm ci --omit=dev --ignore-scripts \
  && npm cache clean --force

FROM node:${NODE_VERSION} AS runtime

ENV NODE_ENV=production \
  PORT=9000 \
  CACHE_SECONDS=86400

WORKDIR /app

RUN addgroup -S readmestats \
  && adduser -S -G readmestats readmestats

COPY --from=build --chown=readmestats:readmestats /src /app

USER readmestats

EXPOSE 9000

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD node -e "const http=require('http');const user=(process.env.WHITELIST||'octocat').split(',')[0];http.get('http://127.0.0.1:'+(process.env.PORT||9000)+'/api?username='+encodeURIComponent(user),res=>process.exit(res.statusCode<500?0:1)).on('error',()=>process.exit(1));"

CMD ["node", "express.js"]
