# 'dev' or 'runtime' container build
ARG ARG_VARIANT=dev
# Use an official Python base image from the Docker Hub
FROM python:3.10-slim AS autogpt-base

# Install browsers
RUN apt-get update && apt-get install -y \
    chromium-driver firefox-esr ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install utilities
RUN apt-get update && apt-get install -y \
    sudo curl jq wget git \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV PIP_NO_CACHE_DIR=yes \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Setting ARGs and ENVs for user creation and workspace setup
ARG ARG_USERNAME="app"
ARG ARG_USER_UID=1000
ARG ARG_USER_GID=$ARG_USER_UID
ARG ARG_APP_ROOT="/app"
ENV USERNAME $ARG_USERNAME
ENV USER_UID $ARG_USER_UID
ENV USER_GID $ARG_USER_GID
ENV APP_ROOT $ARG_APP_ROOT

# Creates a non-root user with sudo privileges
RUN groupadd --gid $USER_GID $USERNAME \
    && adduser --uid $USER_UID --gid $USER_GID --force-badname --disabled-password --gecos "" $USERNAME  \
    && echo "$USERNAME:$USERNAME" | chpasswd \
    && adduser $USERNAME sudo \
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers \
    && mkdir -p /etc/sudoers.d \
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# Switches to the newly created user
USER $USERNAME
# Sets up the workspace for the user
RUN sudo rm -rf $APP_ROOT && sudo mkdir -p $APP_ROOT
RUN sudo chown -R $USERNAME:$USERNAME $APP_ROOT

# Set the PATH to include the user's local bin directory
ENV PATH="$PATH:/home/$USERNAME/.local/bin"

# Declare working directory
WORKDIR $APP_ROOT

# Install the required python packages globally
COPY src/requirements.txt .

# Set the entrypoint
ENTRYPOINT ["python", "-m", "autogpt", "--install-plugin-deps"]

# dev build -> include everything
FROM autogpt-base as autogpt-dev
USER $USERNAME
RUN sudo pip install --no-cache-dir -r requirements.txt
ONBUILD COPY . ./

# runtime build -> include bare minimum
FROM autogpt-base as autogpt-runtime
USER $USERNAME
RUN sed -i '/Items below this point will not be included in the Docker Image/,$d' requirements.txt && \
    sudo pip install --no-cache-dir -r requirements.txt
ONBUILD COPY src/autogpt/ ./autogpt
ONBUILD COPY src/scripts/ ./scripts
ONBUILD COPY src/plugins/ ./plugins
ONBUILD COPY src/prompt_settings.yaml .
ONBUILD RUN mkdir ./data

FROM autogpt-${ARG_VARIANT} AS auto-gpt
