
ARG VERSION=3.9-slim
FROM python:$VERSION

# You would typically obtain this latest version from an API endpoint and use that for the runner version
# as the runner will self update to the latest version when it gets its first job.
# The reason this is specified is so that we can test the upgrade scenarios using this Dockerfile.
ARG GH_RUNNER_VERSION=2.296.1

ARG COMMIT_SHA=main
ARG CREATED=""

ENV RUNNER_NAME=""
ENV GITHUB_SERVER=""
ENV GITHUB_TOKEN=""
ENV RUNNER_LABELS=""
ENV RUNNER_OPTIONS=""
ENV RUNNER_WORK_DIRECTORY="_work"
ENV RUNNER_ALLOW_RUNASROOT=true
ENV AGENT_TOOLS_DIRECTORY=/opt/hostedtoolcache

# Fill in your labels as appropriate here
LABEL \
    org.opencontainers.image.created="$CREATED" \
    org.opencontainers.image.authors="Mariano Alesci" \
    org.opencontainers.image.url=https://github.com/malesci/github-actions-runner-container \
    org.opencontainers.image.documentation=https://github.com/malesci/github-actions-runner-container/README.md \
    org.opencontainers.image.source=https://github.com/malesci/github-actions-runner-container \
    org.opencontainers.image.version=$GH_RUNNER_VERSION \
    org.opencontainers.image.revision=$COMMIT_SHA \
    org.opencontainers.image.vendor="Mariano Alesci" \
    org.opencontainers.image.licenses=MIT \
    org.opencontainers.image.ref.name=python-actions-runner \
    org.opencontainers.image.title="GitHub Actions Runner Container - python" \
    org.opencontainers.image.description="GitHub Actions Runner built into a Container to provide self-hosted runners for Enterprise, Organization or Repositories" \
    github_actions_version=$GH_RUNNER_VERSION


# Installs google-chrome, chromedriver
RUN set -ex \
    && apt-get update \
    && apt-get install -y gnupg curl wget unzip ca-certificates --no-install-recommends \
    && wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | \
    gpg --no-default-keyring --keyring gnupg-ring:/etc/apt/trusted.gpg.d/google.gpg --import \
    && chmod 644 /etc/apt/trusted.gpg.d/google.gpg \
    && echo "deb https://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list \
    && apt-get update -y \
    && apt-get install -y google-chrome-stable\
    && CHROME_VERSION=$(google-chrome --product-version | grep -o "[^\.]*\.[^\.]*\.[^\.]*") \
    && CHROMEDRIVER_VERSION=$(curl -s "https://chromedriver.storage.googleapis.com/LATEST_RELEASE_$CHROME_VERSION") \
    && wget -q --continue -P /chromedriver "https://chromedriver.storage.googleapis.com/$CHROMEDRIVER_VERSION/chromedriver_linux64.zip" \
    && unzip /chromedriver/chromedriver* -d /usr/local/bin/

# Install Azure CLI (instructions taken from https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash \
    && echo "azure-cli https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt" >> apt-sources.txt \
    && rm -f /etc/apt/sources.list.d/azure-cli.list \
    && rm -f /etc/apt/sources.list.d/azure-cli.list.save

# Install additional Python requirements
COPY python-actions-runner/requirements.txt .
RUN pip install -r ./requirements.txt

# Set display port as an environment variable
ENV DISPLAY=:99

# Create a user for running actions
RUN useradd -m actions
RUN mkdir -p /home/actions ${AGENT_TOOLS_DIRECTORY}
WORKDIR /home/actions

# jq is used by the runner to extract the token when registering the runner
RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get update \
    && apt-get install jq git -y \
    && curl -L -O https://github.com/actions/runner/releases/download/v${GH_RUNNER_VERSION}/actions-runner-linux-x64-${GH_RUNNER_VERSION}.tar.gz \
    && tar -zxf actions-runner-linux-x64-${GH_RUNNER_VERSION}.tar.gz \
    && rm -f actions-runner-linux-x64-${GH_RUNNER_VERSION}.tar.gz \
    && ./bin/installdependencies.sh \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Copy out the runsvc.sh script to the root directory for running the service
RUN cp bin/runsvc.sh . && chmod +x ./runsvc.sh

COPY common/entrypoint.sh .
RUN chmod +x ./entrypoint.sh

# Now that the OS has been updated to include required packages, update ownership and then switch to actions user
RUN chown -R actions:actions /home/actions ${AGENT_TOOLS_DIRECTORY}

USER actions
CMD [ "./entrypoint.sh" ]
