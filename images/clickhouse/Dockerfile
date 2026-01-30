FROM clickhouse/clickhouse-server:25.5.6

# Install wget to fetch the binary
RUN apt-get update && apt-get install -y wget && rm -rf /var/lib/apt/lists/*

# Fetch histogram-quantile binary
# We fetch both amd64 and arm64 to be safe, or we can use build args.
# For simplicity in this Dockerfile, let's just fetch the one matching the build arch or fetch both and rename.
# Actually, since we are building for specific arch, we can use TARGETARCH if using buildx, or just fetch based on uname.

RUN mkdir -p /var/lib/clickhouse/user_scripts/

# Download for AMD64 (most likely production) and ARM64 (for local M1/M2)
# We'll use a script to detect and download the right one, or just download the one we need.
# Let's download the linux-amd64 one as default for prod.
RUN wget -O /tmp/hq.tar.gz "https://github.com/SigNoz/signoz/releases/download/histogram-quantile%2Fv0.0.1/histogram-quantile_linux_amd64.tar.gz" \
    && tar -xvzf /tmp/hq.tar.gz -C /tmp \
    && mv /tmp/histogram-quantile /var/lib/clickhouse/user_scripts/histogramQuantile \
    && chmod +x /var/lib/clickhouse/user_scripts/histogramQuantile \
    && rm /tmp/hq.tar.gz

# Ensure permissions
RUN chown -R clickhouse:clickhouse /var/lib/clickhouse/user_scripts/
