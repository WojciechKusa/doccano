ARG PYTHON_VERSION="3.8.12-slim-buster"
ARG NODE_VERSION="16.5-alpine3.14"
FROM node:${NODE_VERSION} AS frontend-builder

COPY frontend/ /frontend/
WORKDIR /frontend
ENV PUBLIC_PATH="/static/_nuxt/"

# hadolint ignore=DL3018
RUN apk add -U --no-cache git python3 make g++ \
 && git config --global url."https://github.com/".insteadOf git://github.com/ \
 && yarn install \
 && yarn build \
 && apk del --no-cache git make g++

FROM python:${PYTHON_VERSION} AS backend-builder

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    netcat=1.* \
    libpq-dev=11.* \
    unixodbc-dev=2.* \
    g++=4:* \
    libssl-dev=1.* \
    curl \
 && apt-get clean

WORKDIR /tmp
COPY backend/pyproject.toml backend/poetry.lock /tmp/

# hadolint ignore=DL3013
RUN pip install --upgrade pip \
 && curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python - \
 && PATH="${PATH}:$HOME/.poetry/bin" \
 && poetry export --without-hashes -o /requirements.txt \
 && echo "psycopg2-binary==2.8.6" >> /requirements.txt \
 && echo "django-heroku==0.3.1" >> /requirements.txt \
 && pip install --no-cache-dir -r /requirements.txt \
 && pip wheel --no-cache-dir -r /requirements.txt -w /deps

FROM python:${PYTHON_VERSION} AS runtime

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    libpq-dev \
    unixodbc-dev=2.* \
    libssl-dev=1.* \
 && apt-get clean

RUN useradd -ms /bin/sh doccano

RUN mkdir /data \
 && chown doccano:doccano /data

COPY --from=backend-builder /deps /deps
# hadolint ignore=DL3013
RUN pip install --no-cache-dir -U pip \
 && pip install --no-cache-dir /deps/*.whl \
 && rm -rf /deps

COPY --chown=doccano:doccano . /doccano
WORKDIR /doccano/backend
COPY --from=frontend-builder /frontend/dist /doccano/backend/client/dist
RUN python manage.py collectstatic --noinput
RUN chown -R doccano:doccano .

VOLUME /data
ENV DATABASE_URL="sqlite:////data/doccano.db"

ENV DEBUG="False"
ENV STANDALONE="True"
ENV SECRET_KEY="change-me-in-production"
ENV PORT="8000"
ENV WORKERS="2"
ENV CELERY_WORKERS="2"
ENV GOOGLE_TRACKING_ID=""
ENV AZURE_APPINSIGHTS_IKEY=""
ENV DJANGO_SETTINGS_MODULE="config.settings.production"

USER doccano
EXPOSE ${PORT}

CMD ["/doccano/tools/run.sh"]
