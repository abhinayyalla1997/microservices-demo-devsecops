# Copyright 2024 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Define a default value so it's not empty if the builder fails to provide it
ARG BUILDPLATFORM=linux/amd64

FROM --platform=$BUILDPLATFORM python:3.14.7-slim@sha256:cad9a2c871761c413caa6fdd6441c783451e740a48aaeba60ae62a8b53525ef6 AS base

FROM base AS builder

RUN apt-get -qq update \
    && apt-get install -y --no-install-recommends g++ \
    && rm -rf /var/lib/apt/lists/*

# get packages
COPY requirements.txt .
# setuptools/wheel ship bundled with the base Python image itself, so a
# plain requirements.txt pin doesn't reliably upgrade them (CVE-2025-47273,
# CVE-2026-24049) — upgrade explicitly first.
RUN pip install --upgrade --no-cache-dir "setuptools>=78.1.1" "wheel>=0.46.2" \
    && pip install -r requirements.txt

FROM base
# Enable unbuffered logging
ENV PYTHONUNBUFFERED=1

# Pull current patched OS packages (gzip/pcre2/sqlite3/perl CVEs)
RUN apt-get -qq update \
    && apt-get upgrade -y \
    && rm -rf /var/lib/apt/lists/*

# get packages
WORKDIR /shoppingassistantservice

# Grab packages from builder
COPY --from=builder /usr/local/lib/python3.14/ /usr/local/lib/python3.14/

# Add the application
COPY . .

# set listen port
ENV PORT="8080"
EXPOSE 8080

ENTRYPOINT ["python", "shoppingassistantservice.py"]
