# ============================================
# Stage 1: Builder
# Validasi & minify HTML pakai Python stdlib.
# Stage ini di-buang setelah build.
# ============================================
FROM alpine:3.20 AS builder

RUN apk add --no-cache python3

WORKDIR /build

# Copy source HTML
COPY index.html .

# Validasi: parse HTML, harus ada <html> & <body>
RUN python3 - <<'PY'
import html.parser
import sys

class StrictParser(html.parser.HTMLParser):
    def __init__(self):
        super().__init__()
        self.has_html = False
        self.has_body = False
    def handle_starttag(self, tag, attrs):
        if tag == "html": self.has_html = True
        if tag == "body": self.has_body = True
    def error(self, message):
        raise ValueError(f"HTML parse error: {message}")

p = StrictParser()
with open("index.html") as f:
    p.feed(f.read())

if not p.has_html or not p.has_body:
    print(f"VALIDATION FAILED: html={p.has_html}, body={p.has_body}", file=sys.stderr)
    sys.exit(1)

print("HTML validation OK")
PY

# Minify: hapus komentar, collapse whitespace
RUN python3 - <<'PY'
import re
with open("index.html") as f:
    html = f.read()
html = re.sub(r"<!--(?!\[if)[\s\S]*?-->", "", html)
html = re.sub(r">\s+<", "><", html)
html = re.sub(r"\s{2,}", " ", html)
html = html.strip()
with open("index.min.html", "w") as f:
    f.write(html)
import os
src = os.path.getsize("index.html")
dst = os.path.getsize("index.min.html")
print(f"Minified: {src}B → {dst}B ({100 - dst*100//src}% smaller)")
PY

# ============================================
# Stage 2: Runtime
# Image final — nginx + HTML hasil build.
# ============================================
FROM nginx:1.27-alpine AS runtime

LABEL maintainer="devops@example.com" \
      description="Maintenance page served by nginx" \
      version="1.2.0"

# Hapus default nginx content (index.html bawaan)
# JANGAN hapus default.conf — kita replace isinya dengan nginx.conf kita
RUN rm -rf /usr/share/nginx/html/*

# Copy HTML yang sudah di-minify dari stage builder
COPY --from=builder /build/index.min.html /usr/share/nginx/html/index.html

# Copy nginx config kita (override default server block)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose HTTP port
EXPOSE 80

# Healthcheck pakai endpoint /healthz yang lebih ringan
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost/healthz || exit 1

CMD ["nginx", "-g", "daemon off;"]