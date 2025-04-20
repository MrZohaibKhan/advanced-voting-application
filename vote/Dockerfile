ARG PYTHON_VERSION=3.13.3
FROM python:${PYTHON_VERSION}-alpine

WORKDIR /app

ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}"  \
    appuser

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER appuser

ENTRYPOINT ["python3"]
CMD ["app.py"]